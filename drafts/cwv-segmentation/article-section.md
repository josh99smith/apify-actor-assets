## Segment your field data before you pick a fix

The 75th percentile is one number for your whole audience. It can pass while one slice of that audience has a bad time. A common case: fast desktops and flagship phones, with low-end Android carrying the whole slow tail.

Segmenting also tells you whether the top fix on your triage list is worth doing. A page fails a metric when more than 25% of its visits are over the threshold, and Google scores that over the 28-day CrUX window. So a big lab win only moves the result if it helps enough of your real sessions. If the failing segment is 5% of traffic, fixing it cannot move the p75 on its own. If it is 40%, it can.

*Thanks to [Sujal Khatri](https://dev.to/sujalkhatri) for raising this in the comments.*

**Start with what CrUX already gives you.** CrUX splits data by form factor: phone, tablet and desktop. In PageSpeed Insights that is the `mobile` and `desktop` strategies, so run the triage script for both and see whether a failure belongs to one of them. Country is not available in the CrUX API, the CrUX History API or PageSpeed Insights. It exists only in the CrUX BigQuery dataset, which is origin-level and monthly. Nothing in CrUX separates flagship phones from budget ones, so that needs your own RUM.

**Add device class to your own RUM** with the [`web-vitals`](https://github.com/GoogleChrome/web-vitals) attribution build:

```js
import { onINP, onLCP, onCLS } from 'web-vitals/attribution';

// Rough buckets from navigator.deviceMemory (Chromium only, HTTPS only).
// Values are coarsened to powers of two, and Chrome changed the range in
// version 147 (0.25-8 GB before, 2-32 GB after). These cut-offs work for both.
function deviceClass() {
  const mem = navigator.deviceMemory;
  if (!mem) return 'unknown';
  if (mem <= 2) return 'low';
  if (mem <= 4) return 'mid';
  return 'high';
}

function send(metric) {
  const a = metric.attribution ?? {};
  const body = {
    id: metric.id,
    name: metric.name,
    value: metric.value,
    rating: metric.rating,
    device: deviceClass(),
    net: navigator.connection?.effectiveType ?? null,
    country: document.documentElement.dataset.country ?? null, // set it server-side
    page: location.pathname,
    ...(metric.name === 'INP' && {
      target: a.interactionTarget,
      inputDelay: a.inputDelay,
      processingDuration: a.processingDuration,
      presentationDelay: a.presentationDelay,
    }),
  };
  navigator.sendBeacon('/rum', new Blob([JSON.stringify(body)], { type: 'application/json' }));
}

onINP(send); onLCP(send); onCLS(send);
```

`deviceMemory` and `connection` exist only in Chromium browsers, and `connection.effectiveType` is a coarse estimate. Safari and Firefox sessions land in `unknown`. CrUX is narrower still: Chrome on desktop and Android only, so iPhones, WebView and Edge are not in it. Expect your RUM and CrUX to differ for that reason. Also keep only the last report per `id`, because a metric can report more than once per page view.

**Read p75 per segment, not overall.** Group by page, metric and device class, and compute p75 inside each group. Illustrative numbers, not real data: high-end phones are 55% of sessions with INP p75 140 ms, mid-range 30% at 230 ms, low-end 15% at 610 ms. The overall p75 fails because of the low-end tail. The likely cause is long main-thread tasks that only a slow CPU exposes. Treat any segment with fewer than about 100 samples as noise.

**Choose fixes by impact on the field window.** Add two columns to each failing page in the triage lists:

- Which segment fails, and its share of sessions.
- Whether the top lab opportunity applies to that segment. A lab run is one emulated mid-range phone, so a fix for a problem that only appears on slower CPUs or networks may not show up there.

Start with the failing segment that has the most to gain: its share of sessions times its gap to the threshold. Your own RUM shows the improvement within days. CrUX takes weeks to roll the 28-day window.
