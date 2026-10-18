# Ticketmaster public venue adapter

The `ticketmaster` code-owned adapter reads Ticketmaster's currently public,
undocumented website venue-search endpoint and classifies inventory using the
event page's embedded `#__NEXT_DATA__` JSON. It is **not** the official
Ticketmaster Discovery API, does not use an API key, and is not a general HTTP
or pagination facility. Website behavior can change; unrecognized responses
fail instead of yielding a partial calendar.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "fetch": { "method": "GET" },
      "adapter": {
        "name": "ticketmaster",
        "options": {
          "venueIds": ["123078", "123041"],
          "pageSize": 20,
          "maxRequests": 200,
          "maxEvents": 500
        }
      },
      "parser": {}
    },
    "event": { "type": "None", "input": { "mode": "none" }, "parser": {} }
  },
  "validation": { "minEventsPerFetch": 1 }
}
```

The feed URL remains the source's official `https://www.ticketmaster.com/`
venue or event page URL; no request is made to the feed itself. Supply the
publisher's numeric venue ID(s) in `venueIds`. Options are closed and bounded:
1–10 distinct decimal venue IDs, the currently fixed 20-record `pageSize`, a
whole-run `maxRequests` from 2–1000, and a combined inventory/output `maxEvents`
from 1–5000. Select request/event limits using the actual source inventory,
including detail-page classifications. The adapter does not contain venue IDs
for any particular source.

## Fetch, pagination, and joins

1. For each venue, request only
   `GET https://www.ticketmaster.com/api/search/events/venue?venueId={id}&page={zeroBasedPage}`.
   The host, route, query key order, decimal IDs, HTTPS/default port, and zero-
   based page number are constrained in code. The first page's `total` governs
   exact page count and expected page cardinality (20 except for the last page).
2. Fail the entire execution on changing/invalid totals, malformed records,
   duplicate identities within a venue, missing/extra page records, request or
   event-cap overflow, or non-progress. Identical listings repeated across
   configured venues deduplicate by Ticketmaster's stable 16-hex ID; conflicting
   versions fail.
3. Only events with publisher `dates.dateDisplay == "showDateTime"` and no
   `cancelled`, `postponed`, or `tba` flag are candidates for output. No dates
   are synthesized. Date-only/day-package listings and unscheduled listings
   are excluded rather than converted to midnight performances.
4. Fetch the canonical event page for **every** scheduled candidate, then parse
   only its single embedded `<script id="__NEXT_DATA__">` JSON object. Both
   recognized Ticketmaster page layouts are supported: old `edpData.context`
   event objects and `pageProps.eventInfo`. Detail identity must match the
   inventory ID and a recognized classification array must be present.
   `classifications[].type.name == "Upsell"` excludes an item independent of
   its title. Missing, ambiguous, or malformed layouts/classification fail
   closed; title/prose heuristics are not used.
5. Return nothing until all required detail requests and classification checks
   succeed. Stable IDs are `ticketmaster_{lowercasePublisherId}`. Retry/replay
   re-reads the venue inventory and all required details; no adapter-level
   retries or partial writes are introduced.

All requests are default-port HTTPS to `www.ticketmaster.com` and are limited
to the exact inventory endpoint/query or canonical `/.../event/{16-hex-id}`
paths. The existing adapter transport disables redirects and cookies, filters
unsafe schema-supplied headers, and bounds response bytes/time. Do not add
credentials, API keys, cookies, or private headers to the schema.

## Normalized fields and time semantics

- Title, event URL, date/time, venue, image, and inventory identity come from
  the public venue record. Start and explicit end timestamps must contain an
  offset or `Z`, and are normalized to UTC without reinterpreting publisher
  wall time. Missing end times use the existing generic two-hour estimate;
  this is an estimate, not Ticketmaster-published end-time evidence.
- Venue name, address line, city, and state are projected only from that
  inventory record. No source-wide fallback location is invented.
- Description is plain text from the recognized detail JSON's
  `generalInfo.text`, `webInfoNoHtml`, or `webInfo`, where present. Missing
  publisher descriptions remain null. No marketing prose is synthesized.
- Classification checks use metadata, never event title, text, product labels,
  URL path words, or parent-event add-on references.

This website API is undocumented and may drift. Revalidate representative
payloads after publisher changes. A detail or inventory parse failure is
observable at the adapter stage and fails the whole run before persistence.
For first response, contain repeated work for the affected source, isolate
inventory/pagination/detail-classification/persistence using source and
correlation scopes, repair the contract or adapter against fresh public data,
and replay through the normal worker after validation. Suggested initial alert
baselines are parse success below 90% over 15 minutes/20 fetches, more than
five retries per source in ten minutes, sustained dead-letter growth, or
freshness lag beyond twice the source cadence.
