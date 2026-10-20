---
title: "Your p75 is hiding who's slow: triaging CrUX field failures by device class"
published: false
description: "A passing or failing p75 says nothing about which devices are slow. Split CrUX by form factor, add device class to your own RUM, and pick the fix that moves the 28-day window."
tags: webdev, performance, webperf, javascript
---

This post started as a comment. On the [Core Web Vitals field data vs lab data](https://dev.to/josh99smith/core-web-vitals-field-data-vs-lab-data-why-a-lighthouse-100-can-still-fail-3o26) article, [Sujal Khatri](https://dev.to/sujalkhatri) pointed out that a perfect lab score with zero real taps lets you declare victory while real users stutter. They suggested pairing the field-failure lists with RUM segmented by device class, because p75 hides who is slow.

Here is that idea as a worked example. All numbers below are **illustrative**, not real data.

## Why p75 hides the problem

A page passes a metric when at least 75% of its visits are at or under the threshold. For INP that is 200 ms. That means:

- A page fails when **more than 25%** of visits are over the line.
- A slow segment under 25% of traffic cannot fail the page on its own.
- A slow segment of 40% can.

So the same p75 can come from very different audiences. You need to know which one you have before choosing a fix.

## Step 1: take the split CrUX already gives you

| Source | Form factor | Country | Granularity |
|---|---|---|---|
| CrUX API | phone, tablet, desktop | no | URL or origin, 28-day window |
| CrUX History API | phone, tablet, desktop | no | URL or origin, weekly periods |
| PageSpeed Insights | `mobile` or `desktop` strategy | no | URL, falling back to origin |
| CrUX BigQuery | yes | yes | origin only, monthly |

Run your triage with both strategies first. If a failure only exists on `mobile`, you have already cut the problem in half. Country needs BigQuery, and only at origin level.

CrUX cannot see inside the phone bucket, and it only covers Chrome on desktop and Android. That is why the next step needs your own data.

## Step 2: add device class to your RUM

This is the part of [Sujal Khatri's](https://dev.to/sujalkhatri) advice that CrUX cannot do for you: segmenting RUM by device class. The snippet uses the [`web-vitals`](https://github.com/GoogleChrome/web-vitals) attribution build. The INP fields `interactionTarget`, `inputDelay`, `processingDuration` and `presentationDelay` tell you where the time went.

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

Caveats worth knowing:

- `deviceMemory` and `connection` exist only in Chromium browsers. Safari and Firefox sessions land in `unknown`.
- CrUX is narrower than Chromium: iOS, WebView and Edge are not in it. Your RUM and CrUX will not match exactly.
- A metric can report more than once per page view. Keep the last report per `id`.
- INP only reports when the user interacted, so its shares are shares of reports, not of all page views.

## Step 3: read p75 per segment

Store the beacons and group by page, metric and device class. In Postgres:

```sql
WITH last_report AS (
  SELECT DISTINCT ON (id) *
  FROM rum
  WHERE name = 'INP'
  ORDER BY id, received_at DESC
)
SELECT page, device,
       count(*) AS samples,
       percentile_cont(0.75) WITHIN GROUP (ORDER BY value) AS p75_ms,
       round(100.0 * avg((value > 200)::int), 1) AS pct_over_200
FROM last_report
GROUP BY page, device
ORDER BY page, p75_ms DESC;
```

Ignore any row under about 100 samples. Here is what one page could look like:

| Device class | Share of reports | INP p75 | Over 200 ms |
|---|---|---|---|
| high | 55% | 140 ms | 8% |
| mid | 30% | 230 ms | 40% |
| low | 15% | 610 ms | 85% |
| **all** | 100% | over 200 ms | **29%** |

The overall number fails because 29% is above the 25% line. Two segments are failing, and the overall p75 does not tell you that. The likely cause is long main-thread tasks, which only a slow CPU exposes.

## Step 4: pick the fix by impact on the field window

For each failing segment, score it as **share of sessions × gap to the threshold**:

- Low: 15% × (610 − 200) = **61.5**
- Mid: 30% × (230 − 200) = **9**

Low-end goes first. Then ask the second question: does the top lab opportunity apply to that segment? Lighthouse runs one emulated mid-range phone, so:

- If your triage list says the top fix is "reduce JavaScript execution time", it plausibly applies to the low-end tail.
- If the segment is slow because of its network or region, a lab opportunity may help the wrong people.

Last, check that the fix can reach the line. You need the overall share over 200 ms below 25%, from 29% today. Getting the low-end segment from 85% over to 50% over takes the total to about 24%, which passes. Getting the mid segment from 40% to 25% takes it to just under 25%, which also passes, with a much smaller change. The score ranks upside, not effort, so if the low-end fix needs a rewrite, the mid-range segment may be the cheaper way over the line.

Your own RUM shows the effect within days. CrUX takes weeks, because it scores a 28-day window.

## Where the field-failure list comes from

If you want a list of failing URLs to start from, [PageSpeed Insights Audit](https://apify.com/josh99smith/pagespeed-insights-audit) is an open-source Apify Actor that returns one flat record per URL and strategy, with `fieldData` (including `originFallback`) and, with `includeAuditDetails`, the lab `opportunities`. The triage script in the [original article](https://dev.to/josh99smith/core-web-vitals-field-data-vs-lab-data-why-a-lighthouse-100-can-still-fail-3o26) sorts it for you. It costs $0.004 per successfully audited URL and strategy, and failed audits are free.

Disclosure: I built this Actor. Source: [github.com/josh99smith/pagespeed-insights-audit](https://github.com/josh99smith/pagespeed-insights-audit). PageSpeed Insights and Lighthouse are Google trademarks; the Actor is not affiliated with Google.
