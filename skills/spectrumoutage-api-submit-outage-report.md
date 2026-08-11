---
name: Submit a Spectrum outage report
description: Submit a community outage report to SpectrumOutage.us on behalf of a user, with the correct enum values, and confirm it landed — including the duplicate risk that comes from an API with no idempotency contract.
api: openapi/spectrumoutage-api-openapi.yml
operations:
  - createReport
  - getZipStatus
  - listReports
generated: '2026-08-11'
method: generated
source: openapi/spectrumoutage-api-openapi.yml
---

# Submit a Spectrum outage report

`createReport` is the only write operation in this API. It writes to a public
community dataset, so treat it as a consequential action: confirm with the user before
calling it, and never submit a report the user did not ask for.

## Before you start

- `POST https://spectrumoutage.us/api/v1/reports`
- `Authorization: Bearer YOUR_API_KEY` and `Content-Type: application/json`
- With an API key there is no reCAPTCHA step.

## Collect and validate the input

The body is `CreateReportInput`. Three fields are required and two of them are closed
enums — a value outside the enum is a `400 INVALID_INPUT`, so validate locally first
rather than letting the API reject it.

| Field | Required | Accepted values |
|---|---|---|
| `zipCode` | yes | a 5-digit US ZIP, as a string |
| `serviceType` | yes | `internet`, `tv`, `phone`, `all` |
| `startedWhen` | yes | `just_now`, `last_hour`, `few_hours`, `today`, `yesterday`, `days_ago` |
| `note` | no | free text, at most 500 characters |

Map the user's phrasing onto the enum rather than inventing a value: "my internet has
been out since this morning" is `serviceType: internet`, `startedWhen: today`. If the
user's description does not fit a bucket, pick the nearest coarser one and say so.

Never put personally identifying information in `note` — it is published to a public
dataset. Strip names, addresses beyond the ZIP, account numbers and phone numbers.

## Steps

1. **Check the ZIP first.** Call `getZipStatus` (`GET /zip/{zip}`). This validates the ZIP
   and tells you whether the outage is already reported (`data.status = issues_reported`).
   If it is, tell the user — they may not need to file at all.
2. **Confirm with the user.** Read back the ZIP, service and start bucket you are about to
   submit. Get an explicit yes.
3. **Submit once.** Call `createReport`. A `201` returns `ReportResponse` — read
   `data.id`, `data.createdAt` and echo them back as the receipt.
4. **Do not retry blindly.** There is no idempotency key and no request deduplication on
   this endpoint, so a retried POST creates a *second* report and inflates the count for
   that ZIP. On a timeout or an ambiguous failure, call `listReports`
   (`GET /reports?page=1&pageSize=50`) or `getZipStatus` and look for your report by
   `zipCode` + `serviceType` + `createdAt` before sending anything again.
5. **Verify.** `getZipStatus` on the same ZIP should now include the new report in
   `data.reports[]`.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 400 `INVALID_INPUT` | Bad body — usually a value outside an enum, a malformed ZIP, or a note over 500 chars | Fix the field, resubmit once |
| 401 | Missing `Authorization` header | Add it; do not retry without it |
| 403 | Invalid API key | Stop — the key is wrong |
| 429 | Over 100 req/min per key | Back off. No `Retry-After` header is sent |
| 500 `STATS_UNAVAILABLE` | Server error | Check `listReports` before retrying, to avoid a duplicate |

## Attribution

Link back to `https://spectrumoutage.us` when displaying data from this API. The tracker
is independent and not affiliated with Charter Communications or Spectrum.
