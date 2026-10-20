# Notes for Josh

## Status: nothing is published

- **Not posted.** You asked mid-session to post the update to the article. I couldn't: this environment has no dev.to API key or dev.to connector (an unauthenticated call to the dev.to API returns 401). Everything is a draft.
- **Fastest way to post it:** `article-with-section.md` is your current live article body (fetched from dev.to) with the new section already inserted before "Cost and limits". Paste it into the editor, or insert `article-section.md` by hand.
- Editing the article by API would replace the whole body, so I'd want you to approve that first. If you want me to do it, add a dev.to API key as an environment secret (not in chat).

## Corrections to the starting draft

| # | Draft said | Now says | Source |
|---|---|---|---|
| 1 | `deviceMemory` "capped at 8" | Values are coarsened powers of two; **range changed in Chrome 147** (0.25-8 GB before, 2-32 GB after). Cut-offs in the snippet work for both. | MDN browser-compat data, MDN `deviceMemory` page |
| 2 | "Chromium browsers, which is also the population CrUX measures" | CrUX is **Chrome only** (desktop + Android). It excludes iOS Chrome, WebView and Edge. | CrUX methodology page |
| 3 | "desktop and recent iPhones are fast" | Changed to "fast desktops and flagship phones". iPhones are not in CrUX at all. | CrUX methodology page |
| 4 | "fixing a 5% segment won't move the p75" | Made precise: a metric fails when **more than 25%** of visits are over the threshold, so a 5% segment can't flip it alone. | p75 definition (CrUX API docs) |
| 5 | Country "needs BigQuery" | Kept, and added that BigQuery is **origin-level and monthly**. Country is absent from the CrUX API, History API and PSI. | CrUX API, History API, BigQuery docs |
| 6 | Snippet fell back to `hardwareConcurrency` | Removed. Non-Chromium sessions are `unknown`, so you don't mix guessed buckets into the table. | My call, see below |
| 7 | Snippet sent `interactionTarget` only | Added `inputDelay`, `processingDuration`, `presentationDelay` for INP. Added `id` so you can keep the last report per page view. Sends JSON via `Blob`. | web-vitals README |
| 8 | "Fix the largest failing segment first" | Ambiguous. Now "most to gain: share of sessions × gap to threshold". | See below |

## Verified as correct

- `web-vitals/attribution` import path, and INP fields `interactionTarget`, `inputDelay`, `processingDuration`, `presentationDelay` (web-vitals README).
- CrUX API and History API: form factor only (`PHONE`, `TABLET`, `DESKTOP`), no country. History API is weekly periods of 28 days.
- INP thresholds 200 ms / 500 ms, and TBT as a proxy but "not a substitute" (web.dev).
- `navigator.connection`: Chrome 61+, no Safari, removed from Firefox. `deviceMemory`: Chrome 63+, no Safari or Firefox, HTTPS only.

## Not fully verified

- **Chrome 147 value change** comes from MDN's compat data. I couldn't load the W3C spec text to confirm it. Worth a 30-second look at the MDN `deviceMemory` page before publishing.
- **"PSI mobile strategy = phone form factor"** is how PSI presents it, but the PSI API docs don't say it in so many words. I wrote it as "that is the `mobile` and `desktop` strategies".
- **"Lighthouse runs one emulated mid-range phone."** True of Lighthouse's mobile defaults as I know them. I did not re-check the current device name.

## Judgement calls to look over

- **Scoring rule.** Share × gap ranks *upside*, not effort. In the worked example it picks the low-end segment, but the mid segment (30 ms over) is the cheaper way past the 25% line. The post says so. Cut that paragraph if it's too much.
- **Table numbers are illustrative**, labelled as such in both files. The "over 200 ms" column is mine (the 29% total is just the share-weighted sum: 29.15%).
- **Attribution check:** the comment page shows the INP point both from `omyvnss` and from Sujal, who repeated it. The brief credits Sujal for both points. I credited Sujal for the segmentation advice and the comment as a whole, and did not credit `omyvnss`. Consider crediting them too, since their comment came first.
- **Your earlier reply** to Sujal already says "I'll add a short section on this". `comment-reply.md` is a follow-up that says it's now done.
- **Voice.** First person appears only in the Actor disclosure, as in the original. The comment reply is first person because it's yours.
- **Actor claims** stick to the original article: flat record per URL and strategy, `fieldData.originFallback`, `opportunities` via `includeAuditDetails`, $0.004 per successful audit, failed audits free. No phone-call CTAs anywhere.

## Open items for you

1. Run the RUM snippet on one of your own pages for about a week, then replace the illustrative numbers with real ones (or keep the "illustrative" label).
2. Target publish date for the follow-up: **Tue Nov 3, 2026** (checked: it is a Tuesday). Flip `published: false` yourself.
3. Insert the new section into the article yourself, and post `comment-reply.md` yourself.
4. After the follow-up is live, add its link to the end of the new article section and to the optional last line of the reply.
5. dev.to tag limit is 4. The follow-up uses exactly `webdev, performance, webperf, javascript`.
