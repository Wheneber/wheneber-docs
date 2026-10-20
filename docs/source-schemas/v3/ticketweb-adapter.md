# TicketWeb WordPress calendar adapter

`ticketweb` supports the public **TicketWeb Event Discovery WordPress plugin,
calendar2** protocol. It is not a general TicketWeb API client, JavaScript
interpreter, or arbitrary request workflow. No hostname, show title or event ID
is built into the adapter.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "fetch": {
        "method": "GET",
        "requestHeaders": {
          "User-Agent": "Wheneber/1.0 (+https://dev.app.wheneber.com)"
        }
      },
      "adapter": {
        "name": "ticketweb",
        "options": {
          "timeZone": "America/Los_Angeles",
          "allowedOrigins": ["https://calendar.example"],
          "futureMonths": 12,
          "maxRequests": 13,
          "maxEvents": 1000,
          "venue": {
            "name": "Example Venue",
            "address": "123 Example St",
            "evidenceUrl": "https://calendar.example/"
          }
        }
      },
      "parser": {}
    },
    "event": { "type": "None", "input": { "mode": "none" }, "parser": {} }
  },
  "validation": { "minEventsPerFetch": 1 }
}
```

Use the publisher's calendar page as the source feed URL. The optional `venue`
above is illustrative, not a fact suitable for a real source. Verify a
single-venue calendar before supplying it. A missing published venue without
this evidence-backed fact fails extraction. Do not infer event location from
unrelated sidebar/contact text.

## Execution and bounds

1. GET the configured page with the normal shared header filter. Identify
   exactly one inline public `get_events_for_calendar` configuration. Extract
   bounded JS **string literals** for endpoint, fresh public nonce, JSON params,
   and standard month view; never execute JavaScript. Multiple calendars or
   unsupported literal/config shapes fail explicitly.
2. Preserve the publisher's calendar/filter/presentation params. Replace stale
   month/year/current_datetime values using the current source-local time.
   Reject authentication-like params and inventory-suppressing limit/pagination,
   first-only, one-per-date or max-per-venue settings rather than silently
   discarding rows. Only page zero/no pagination is supported.
3. POST the public action and fresh nonce as form fields to the advertised,
   same-origin `.../wp-admin/admin-ajax.php`, one local calendar month
   per request using `[month start, next-month start)`. The publisher compares
   dates with an exclusive end, ignoring the supplied end time: using last-day
   23:59:59 would silently lose that day's shows. Include the advertised standard-view/calcount/filter/mobile-list
   fields. FormUrlEncodedContent encodes the preserved params exactly once.
4. Join `events` rows to `popupdata` by the `tw-event-dialog-{publisherId}`
   fragment target. The row is the schedule inventory; `mobile_events` and
   mobile list markup are alternate presentations, not additional occurrences.
   Missing/extra/duplicate popups, conflicting duplicate occurrences, or invalid
   required details fail the whole run before persistence. Equal duplicate rows
   deduplicate.
5. Return normalized occurrences through the existing adapter registry. Runtime
   and schema test-fetch use the same prepared adapter, transport, header policy
   and field projection. An unchanged bootstrap does not skip AJAX extraction.

`futureMonths` defaults to **12 including the current month** (range 1–24).
For example, a run in October requests October through September of next year;
it does not start today or include the following October. Empty months do not
stop discovery. `maxRequests` defaults to months + 1 (range 2–25) and must cover
the entire window. `maxEvents` defaults to 1000 (range 1–5000). Schema/server event
caps, response byte limits, timeout and cancellation fail rather than truncate.
The shared freshness filter runs after extraction and minimum-event validation.

Every requested URL must be HTTPS, without credentials or fragments, on both
the exact feed origin and `allowedOrigins`. Bootstrap and AJAX endpoint queries
are forbidden. Cookies, redirects and schema-supplied Authorization/X-WP-Nonce
are disabled by the existing adapter transport. The fresh public nonce goes
only in the publisher-advertised form, not an auth header. Do not save a live
nonce, request body or response bootstrap in diagnostics.

## Field semantics and evidence

- `start` date + `displayTime` are show start, **not `doors`**. Cross-check
  `sortkey` when present, and require matching popup date/time. Dates include
  an explicit year. Publisher-local wall times are converted using configured
  `timeZone`; DST ambiguity/nonexistent times fail instead of guessing.
- The observed plugin does not publish end times. The [shared policy](../schedule-policy.md)
  supplies **start + one elapsed hour**, including midnight/DST crossings, and
  appends a disclosure once after enrichment. Explicit local `end` in
  `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-ddTHH:mm:ss` is preserved and validated.
- A verified date with no row or popup clock is all-day using source-local
  midnight and next midnight. Date-only `end` is exclusive; an end clock
  without a start clock fails. The popup date must still corroborate the row.
- Identity is `ticketweb_{publisherId}_{yyyyMMddTHHmmss}` in source-local time.
  Row order, title edits and UTC DST offsets do not affect it. Different
  publisher IDs or scheduled occurrences remain distinct. Conflicting popup
  schedules for the same ID are unsupported and fail rather than guess.
- Title comes from the row; the joined popup must also have a title, but its
  display typography may differ (observed plugin replacement-character damage).
  Identity and schedule, not title text, are authoritative join evidence. Description is
  sanitized popup text (the plugin may publish only an excerpt). Image is the
  popup's full-size image when present. Optional description/image gaps are
  allowed; never fabricate enrichment.
- `SourceUrl` is the authoritative same-origin popup title permalink, not
  `#tw-event-dialog-...`. The popup's TicketWeb purchase link is separate and
  is not substituted for that permalink. There is no new normalized ticket-URL
  field/API change.
- Published popup venue/address take precedence. An optional verified
  single-venue configuration fills blanks only for that venue.

## Failure response

Fetch diagnostics record safe host/path, source/correlation scopes, outcome,
retry count and failure category. Monthly join logs report month and count,
not payloads or nonce. No adapter-level retries are added; a worker retry
re-fetches bootstrap and all months. Stable IDs support normal replay/upsert.

First response: contain repeated work for the affected source; isolate fetch,
bootstrap, join, validation or persistence using source/correlation IDs and safe
categories; repair and locally validate the full window; then authorize replay
through the normal worker. A parse/join failure persists no partial inventory.
Suggested alert baselines (not configured by this change): parse success below
90% over 15 minutes/20 fetches; more than five retries per source in ten minutes;
sustained dead-letter growth; freshness lag beyond twice the source cadence.

Backend support must be deployed before saving an adapter schema. No OpenAPI
generation is needed: this extends source-schema documentation/behavior, not an
HTTP DTO.
