---
name: Map and trend Spectrum outage activity
description: Build a map view or a trend read of Spectrum outage activity from map pins, the paginated report feed, and the national dashboard — with correct pagination and a real freshness statement.
api: openapi/spectrumoutage-api-openapi.yml
operations:
  - getMapData
  - listReports
  - getStats
  - listStates
generated: '2026-08-11'
method: generated
source: openapi/spectrumoutage-api-openapi.yml
---

# Map and trend Spectrum outage activity

For dashboards, visualizations and "is this getting worse?" questions, rather than a
single-location lookup.

## Before you start

- Base URL `https://spectrumoutage.us/api/v1`, `Authorization: Bearer YOUR_API_KEY`.
- 100 requests per minute per key, no rate-limit headers. A map refresh loop will burn
  that budget quickly — cache and poll no faster than you need.

## Steps

1. **Pull the map layer.** Call `getMapData` (`GET /map?range=1h`). `range` accepts
   exactly `1h` (default), `24h` or `48h`; anything else is a `400`. The echoed
   `meta.range` confirms which window you actually got — read it rather than assuming.
   `data` holds clusters, pins and a summary and is declared as an untyped object in the
   specification, so probe the returned shape defensively and do not hard-code field paths.

2. **Pull the report feed for the detail layer.** Call `listReports`
   (`GET /reports?page=1&pageSize=50`), newest first. `pageSize` maxes out at 100.
   Paginate on the `meta` envelope: `page`, `pageSize`, `total`, `totalPages`. Stop when
   `page` reaches `totalPages` — do not keep incrementing until you get an empty page.
   Each row is an `OutageReport`: `id`, `zipCode`, `serviceType`, `startedWhen`, `note`,
   `createdAt`.

3. **Pull the national aggregate for the headline.** Call `getStats` (`GET /stats`):
   `reportsLast30Min`, `trend` (`up` | `down` | `stable`), `reportsToday`,
   `reportsThisMonth`, `citiesAffected`, `status`, an hourly `timeline[]` of
   `{hour, count}`, a `breakdown` by service type, `topAreas[]` of `{zipCode, count}`,
   and `lastUpdated`.

4. **Add the state choropleth.** Call `listStates` (`GET /states`) for a slug-to-count map
   across all US states. It is a free-form map with no declared schema — iterate its keys,
   do not expect a fixed set.

5. **State freshness honestly.** Put `data.lastUpdated` from `getStats` and the `meta.range`
   from `getMapData` on the view. Counts are community reports, not measured outages, and
   they are biased toward places with more reporters — say so on any visualization.

## Budgeting your calls

A single full refresh is 4 calls (`getMapData`, `listReports`, `getStats`, `listStates`)
plus one extra per additional report page. At 100 req/min per key, a 50-report page size
and a 60-second refresh is comfortable; a 10-second refresh with deep pagination is not.

## Error handling

`400` on a bad `range` or page parameter, `401` with no bearer header, `403` on a bad key,
`429` over the limit with no `Retry-After` to read, `500 STATS_UNAVAILABLE` on a
server-side data load failure. Errors are `{"error": {"message": "...", "code": "..."}}`
and `code` is absent on 401/403 — branch on HTTP status first.

## Attribution

Link back to `https://spectrumoutage.us` on any published view. Independent tracker, not
affiliated with Charter Communications or Spectrum.
