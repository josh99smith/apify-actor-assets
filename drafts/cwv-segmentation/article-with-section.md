---
title: "Core Web Vitals field data vs lab data: why a Lighthouse 100 can still fail"
published: true
description: "Why Lighthouse lab scores and Chrome UX Report field data disagree, plus a script that lists the pages failing Core Web Vitals in real-user data first."
tags: webperf, seo, webdev, corewebvitals
cover_image: https://raw.githubusercontent.com/josh99smith/apify-actor-assets/main/banners/pagespeed-insights-audit.png
---

A page scores 100 in Lighthouse, and Search Console's Core Web Vitals report still flags it as failing. Both are right: they measure different things. Here is why they disagree, and a script that sorts a URL list so you fix what Google measures first.

## Lab data and field data

**Lab data** is one simulated load by Lighthouse on Google's test device. It is good for debugging, though scores vary a few points between runs.

**Field data** comes from the Chrome UX Report (CrUX): real Chrome users, a trailing 28-day window, reported at the 75th percentile, so three in four visits were at least that good.

Thresholds from web.dev's Web Vitals guidance, judged at the 75th percentile:

- LCP (loading): good at 2.5 seconds or less, poor above 4 seconds.
- INP (responsiveness): good at 200 ms or less, poor above 500 ms.
- CLS (visual stability): good at 0.1 or less, poor above 0.25.

In between is "needs improvement". The PageSpeed Insights API names the buckets `FAST`, `AVERAGE` and `SLOW`. A page passes when all three metrics are good.

## Why a page can score 100 in the lab and still fail

The lab run is one device, one connection, one visit:

- INP cannot be measured in the lab: a Lighthouse load has no user to tap anything. web.dev names Total Blocking Time as the lab stand-in, which is only a proxy.
- Real devices are slower and more varied, and the 75th percentile is the slow end of your audience.
- Layout shift continues after load: late ads, consent banners, lazy images. A lab run does not scroll or wait around.
- Real visits differ: warm caches, logged-in views, A/B variants, other countries.
- Field data covers 28 days, so a fix shipped yesterday barely moves it.

It works in reverse too: a page scoring 60 in the lab can pass in the field. That is a lab-only problem, second in the queue.

## Page-level data or origin fallback

PageSpeed Insights uses data for the exact URL first. With too few real-user samples it falls back to origin-level data, the combined numbers for the whole site. If the origin has too little as well, there is no field data. The Actor below exposes which level you got as `fieldData.originFallback`.

- `originFallback: false`: the numbers describe this page.
- `originFallback: true`: origin-level numbers, identical for every page of that origin that fell back.
- `fieldData: null`: no usable field data, lab numbers only.

Ten pages failing on fallback are one finding about the site (shared templates, scripts, banners), not ten about pages. A page-level failure is the strongest evidence, so it goes first.

## Get both kinds of data for a list of URLs

The [PageSpeed Insights API v5](https://developers.google.com/speed/docs/insights/v5/get-started) returns both blocks, one URL and one device per call, each taking Google roughly 10 to 30 seconds. A list needs concurrency, retries and quota handling. [PageSpeed Insights Audit](https://apify.com/josh99smith/pagespeed-insights-audit) is an open-source Apify Actor that runs that loop and returns one flat record per URL and strategy.

```json
{
    "urls": ["https://example.com/", "https://example.com/pricing", "https://example.com/blog/post"],
    "strategy": "mobile",
    "categories": ["performance"],
    "includeAuditDetails": true
}
```

Mobile is the strategy Google Search ranks on; `includeAuditDetails` adds `opportunities`. Save as `input.json`:

```bash
curl -X POST "https://api.apify.com/v2/acts/josh99smith~pagespeed-insights-audit/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d @input.json -o results.json
```

For long lists (about 10 minutes per 100 URLs) use the `apify-client` packages.

## A triage script: field failures first

The script calls a page failing when `fieldData.lcpMs`, `inpMs` or `cls` has a `category` other than `FAST`, and skips metrics with no data.

```js
import { readFileSync } from "node:fs";

const rows = JSON.parse(readFileSync(process.argv[2], "utf8"));
const bad = (r) =>
  ["lcpMs", "inpMs", "cls"].filter((k) => {
    const c = r.fieldData?.[k]?.category;
    return c && c !== "FAST";
  });
const sev = (r) => bad(r).reduce((n, k) => n + (r.fieldData[k].category === "SLOW" ? 2 : 1), 0);
const why = (r) =>
  bad(r).map((k) => `${k.replace("Ms", "")} ${r.fieldData[k].percentile} ${r.fieldData[k].category}`).join(", ");
const show = (title, list, fn) => {
  console.log(`\n${title} (${list.length})`);
  list.forEach((r) => console.log(`  ${r.strategy} ${fn(r)}`));
};

const ok = rows.filter((r) => r.success);
const failing = ok.filter((r) => bad(r).length).sort((a, b) => sev(b) - sev(a));
const pageLevel = failing.filter((r) => !r.fieldData.originFallback);
const originLevel = [
  ...new Map(
    failing.filter((r) => r.fieldData.originFallback).map((r) => [r.strategy + new URL(r.url).origin, r])
  ).values(),
];
const labOnly = ok
  .filter((r) => !bad(r).length && (r.scores?.performance ?? 100) < 90)
  .sort((a, b) => a.scores.performance - b.scores.performance);

show("1. FAILING IN FIELD DATA, page level", pageLevel, (r) =>
  `${r.url} ${why(r)} | top fix: ${r.opportunities?.[0]?.title ?? "n/a"}`);
show("2. FAILING AT ORIGIN LEVEL", originLevel, (r) => `${new URL(r.url).origin} ${why(r)}`);
show("3. LAB ONLY", labOnly, (r) =>
  `${r.url} perf ${r.scores.performance}, ${r.fieldData ? "field passes" : "no field data"}`);

const failed = rows.filter((r) => !r.success);
console.log(`\nNot audited (free): ${failed.length} ${failed.map((r) => r.errorType).join(" ")}`);
```

Run `node triage.mjs results.json`. List 1: page-level field failures, most severe first, with the largest lab opportunity as a lead (a lab estimate). List 2: origin-level failures, one row per origin and device. List 3: lab-only, no field failure and performance below 90 (90+ counts as good).

For INP and CLS, which the lab barely sees, Google's `web-vitals` library on your pages shows the interaction or element behind a field number.

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

## Cost and limits

The price is $0.004 per successfully audited URL and strategy. Assuming 150 URLs and every audit succeeds:

- Mobile only: 150 audits, $0.60 per run.
- Mobile and desktop: 300 audits, $1.20 per run.

Failed audits cost nothing and a run stops at the maximum cost you set. Limits:

- Field data needs real traffic: low-traffic pages return `fieldData: null` or origin-level numbers.
- Lab scores vary by a few points between runs; compare moving averages.
- The default Google API key and its daily quota are shared. When it runs out you get free `rate-limited` records; your own free key in `apiKey` gives a dedicated 25,000 audits per day.
- Google must fetch the page: logins, non-public staging and localhost fail with `http-error`. Failures return `success: false` and are unbilled.

If this saves you time, a [review on the Store page](https://apify.com/josh99smith/pagespeed-insights-audit) helps other people find it, and a star on [GitHub](https://github.com/josh99smith/pagespeed-insights-audit) helps too.

Disclosure: I built this Actor. Source: [github.com/josh99smith/pagespeed-insights-audit](https://github.com/josh99smith/pagespeed-insights-audit). PageSpeed Insights and Lighthouse are Google trademarks; the Actor is not affiliated with Google.
