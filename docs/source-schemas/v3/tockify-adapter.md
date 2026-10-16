# Tockify public calendar adapter

Requires a backend revision registering `tockify`. Use the canonical public calendar
page as the feed URL, e.g. `https://calendar.example/sample/`, **not** a fixed-dated
API URL or a subscription URL. Generic parsers are unchanged.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "fetch": {
        "method": "GET",
        "requestHeaders": {"User-Agent": "Wheneber/1.0 (+https://app.wheneber.com)"}
      },
      "adapter": {
        "name": "tockify",
        "options": {
          "futureDays": 365,
          "pageSize": 20,
          "maxRequests": 100,
          "maxEvents": 1000,
          "allowedOrigins": ["https://calendar.example"]
        }
      },
      "parser": {}
    },
    "event": {"type": "None", "input": {"mode": "none"}, "parser": {}}
  },
  "validation": {"minEventsPerFetch": 1}
}
```

## Inventory, schedule and details

Discovery validates the public bootstrap's calendar name, ID, timezone and canonical
URL. The adapter reads the public frontend `/api/ngevent` pinboard protocol with a
rolling `startms`, `start-inclusive=true`, `showAll=false`, and offset increments by
actual returned row count. It exhausts `metaData.hasNext`; neither the bootstrap's
first 20 nor a larger guessed `max` establishes completeness. Page offset drift,
duplicate occurrence identities, non-progress, missing metadata and budget overflow
fail visibly. Inventory beyond the configured horizon is still enumerated, bounded
by the inventory/request caps; only in-window records require detail joins.

Every selected record requires its public detail bootstrap with exactly one matching
calendar/occurrence. Detail epoch milliseconds, interpreted using
`DateTimeOffset.FromUnixTimeMilliseconds`, are authoritative, not an ICS schedule or
doors clock in prose. A publisher ICS record can differ from its public showtime.
Timezone identifiers are checked but never used to reinterpret epoch instants.
Zero-duration, all-day and multi-day/range events retain published boundaries.
Negative durations, missing dates/timezones and unsupported statuses fail. Published
cancelled/canceled/deleted occurrences are excluded. There is no invented default end,
date inference or recurrence expansion; already expanded instances use upstream
calendar ID plus singleton `uid`, or repeating `uid/rid/tid`. The publisher's repeating
occurrence token `eid.tid` is not the mutable `when.start.millis`; `seq` is deliberately
not an identity component. Repeat detail paths include `/rid`. Unknown kinds or ambiguous
duplicate identities fail rather than adding a current-date/index suffix. Validate real
recurring inventory before enabling a new source; rekeying upstream occurrence tokens
requires reconciliation, not a claim that IDs survive arbitrary publisher series rewrites.

The output identity excludes `sorter`, current start, ticket URL and array position.
Canonical public detail URLs replace potentially schemeless custom ticket links.
Description HTML becomes plain text with scripts/styles/media widgets removed.
Images use published absolute HTTP(S) OpenGraph URLs; image URLs are metadata only,
never fetched by this adapter. Missing optional description/image/location logs counts,
not a whole-run failure. Missing location uses the existing `TBD` convention.

Optional `venue` has exactly `name`, `address`, `evidenceUrl` (nonblank, max 500 chars).
Use only an independently verified single-venue source fact; evidence must be its
same-origin public calendar page. Event `where.place/address` takes precedence; a
different published venue cannot be relabelled. Do not infer an event venue from
contact/office chrome or geocode placeholders.

## Bounds and access

All five options shown are required; unknown keys fail closed. `futureDays`: 1–730;
`pageSize`: 1–100; `maxRequests`: 2–1000; `maxEvents`: 1–5000;
`allowedOrigins`: 1–10 unique default-port HTTPS origins. The shared effective event
cap also applies. Every request must match the feed origin and code-owned calendar,
detail or exact API query rules. Canonical host changes require operator selection
of the new feed, not an automatic host switch. Shared timeout/byte limits, cancellation,
restricted header filtering, no cookies/redirects and safe host/path-only fetch logs
apply equally to worker and test-fetch.

This is a public read-only frontend protocol, not an authenticated API integration.
An indexing `robots.txt` exclusion is distinct from authentication/access controls.
Operators must confirm appropriate publisher use; do not bypass HTTP 401/403, WAF,
challenges, private calendars or publisher restrictions. No tokens, cookies, proxy,
browser impersonation or alternate-host evasion. The adapter never retries internally.

## Validation and first response

Compare the entire configured window against an independent inventory (an advertised
ICS subscription plus independently decoded public details is useful). Check exact
start/end, IDs, images, descriptions, canonical URLs and venue provenance, not only
counts. Test-fetch samples omit some detail fields; retain a local full sample.
No event list reaches persistence until every required inventory/detail request passes.
Freshness is end >= now minus 24 hours; starts at or beyond the future horizon are
excluded. A minimum is evaluated through the existing pipeline.

Contain repeated failed runs; isolate `discover`, `inventory`, `detail_join` or
`quality` with SourceId/CorrelationId scopes and shared fetch diagnostics. Recover
by correcting configuration/publisher drift, validating locally and test-fetching,
then running one separately authorized replay. Existing snapshot IDs make same-ID
replay idempotent; migration from old sorter IDs needs explicit reconciliation.
Suggested alerts (not automatically provisioned): any required-stage failure or
parse success below 99% over 20 runs, >=3 retries/source in 15 minutes, any dead-letter
growth, freshness lag >2 scheduled source intervals. Optional-field warning counts
are quality signals, not reasons for blind retry.
