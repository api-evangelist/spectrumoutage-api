---
name: Check Spectrum outage status for a location
description: Determine whether Spectrum internet, TV, or phone service is disrupted at a ZIP code, city, or state using the SpectrumOutage API, and summarize the current national picture.
api: openapi/spectrumoutage-api-openapi.yml
operations:
  - getZipStatus
  - getCity
  - listStates
  - getState
  - getStats
generated: '2026-08-11'
method: generated
source: openapi/spectrumoutage-api-openapi.yml
---

# Check Spectrum outage status for a location

Answers "is Spectrum down here?" from community-reported outage data. The data is
crowdsourced from user reports, not from Charter Communications — never present it as
an official Spectrum status.

## Before you start

- Base URL: `https://spectrumoutage.us/api/v1`
- Every request needs `Authorization: Bearer YOUR_API_KEY`. There is no anonymous mode
  and no self-service key issuance — a key is obtained by emailing the provider.
- Rate limit: 100 requests per minute per key. There are no `RateLimit-*` headers and no
  `Retry-After`, so track your own call budget rather than reading one off the response.
- Responses are wrapped: `{"data": {...}, "meta": {...}}`. Read the payload from `data`.

## Steps

1. **If you have a 5-digit US ZIP, use it — it is the most precise view.**
   Call `getZipStatus` (`GET /zip/{zip}`). Read `data.status`, which is exactly one of
   `issues_reported` or `all_clear`, and `data.reports[]` for the individual reports
   behind that verdict. Each report carries `serviceType`
   (`internet` | `tv` | `phone` | `all`), `startedWhen`, and `createdAt`.
   - A `400` means the ZIP is not a valid 5-digit US ZIP. Re-ask the user rather than retrying.

2. **If you have a city instead, use its slug.**
   Call `getCity` (`GET /cities/{slug}`) with a lowercase hyphenated slug —
   `los-angeles`, `new-york-city`, `charlotte`, `dallas`. A `404` with
   `error.code = CITY_NOT_FOUND` means the slug is wrong; do not guess a second slug more
   than once, fall back to step 3 or to the ZIP lookup.
   - `data` is declared as an untyped object in the specification. Do not assume field
     names; read what is actually returned and degrade gracefully if a field is missing.

3. **If you have a state, or need to find which states are worst hit.**
   Call `listStates` (`GET /states`) for a slug-to-report-count map across all US states,
   then `getState` (`GET /states/{slug}`) for one state — `california`, `texas`,
   `new-york`. `404` with `error.code = STATE_NOT_FOUND` means a bad slug.

4. **Add national context when the answer is ambiguous.**
   Call `getStats` (`GET /stats`). Use `data.reportsLast30Min` and `data.trend`
   (`up` | `down` | `stable`) to say whether the problem is growing, `data.status`
   (`all_clear` | `issues`) for the national verdict, `data.breakdown` for which service
   is affected, and `data.topAreas` for the worst ZIPs. `data.lastUpdated` tells you how
   fresh the aggregate is — quote it.

5. **Report the answer with its limits.**
   Say how many reports the verdict rests on and how recent they are. A ZIP with two
   reports is not an outage. `startedWhen` is a coarse bucket
   (`just_now`, `last_hour`, `few_hours`, `today`, `yesterday`, `days_ago`), not a
   timestamp — never convert it into a precise start time.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 | Missing `Authorization` header | Add the bearer header; do not retry without it |
| 403 | Invalid API key | Stop. The key is wrong — retrying will not fix it |
| 400 `INVALID_INPUT` | Bad ZIP or parameter | Fix the input, ask the user again |
| 404 `CITY_NOT_FOUND` / `STATE_NOT_FOUND` | Unknown slug | Try `listStates`, or switch to the ZIP lookup |
| 429 | Over 100 req/min | Back off with your own delay — no `Retry-After` is sent |
| 500 `STATS_UNAVAILABLE` | Server-side data load failed | Retry once after a delay |

Errors come back as `{"error": {"message": "...", "code": "..."}}`. `code` is absent on
401 and 403, so branch on the HTTP status first and only then on `error.code`.

## Attribution

When you display this data, link back to `https://spectrumoutage.us`. The provider
requires attribution and the data is community-sourced and provided as-is. Always state
that the tracker is independent and not affiliated with Charter Communications or Spectrum.
