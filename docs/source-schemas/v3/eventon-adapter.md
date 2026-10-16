# EventON source adapter

Select the compiled, allowlisted adapter with
`pipeline.calendar.adapter.name: "eventon"` and its `options`. There is no assembly,
class-name, script, or arbitrary plugin loading. Omitting `adapter` preserves the
existing JSON/HTML/ICS/RSS execution paths. Unknown adapters and options fail.

Use schema version 3, calendar type `Html`, an empty calendar `parser: {}`, and
event stage `{"type":"None","input":{"mode":"none"},"parser":{}}`. Adapter execution
owns projection and monthly traversal: do not configure generic parser mappings,
authentication, pagination, routing, or detail stages alongside it.
`fetch.method`, if supplied, must be `GET` (bootstrap); monthly read-only POSTs
are adapter-owned, not generic schema-defined write operations.

Required options:

| Option | Meaning |
|---|---|
| `futureMonths` | Explicit integer 1–24, **including the current source-local month**. No default; ask the source owner. |
| `timeZone` | Supported timezone, preferably IANA, for exact local month boundaries. |
| `allowedOrigins` | 1–10 explicit HTTPS origins; all adapter URLs must additionally equal the feed origin including port. |
| `maxRequests` | Integer 2–25, at least `futureMonths + 1`. One bootstrap GET plus one POST per month. |
| `maxEvents` | Whole-window unique occurrence limit 1–5000, also bounded by the server limit. Overflow fails without publishing partial data. |

Optional `venue: {name,address,evidenceUrl}` is an explicit **source-wide fact**
for a verified single-venue calendar. Record official provenance when authoring
it. The adapter prefers event-card location, then this fact; missing both fails,
never `TBD`. Do not use this to hide multi-venue data loss. Evidence URLs must be
same-origin HTTPS; they are provenance, not automatically fetched assertions.
Descriptions come only from the matching card's `.eventon_desc_in`, never the
page sidebar, venue biography, or a fallback string.

Each invocation GETs the public calendar page and refreshes `data-sc` and public
`evo_general_params` nonces in memory. The discovered `evo_ajax_url` template
must pass origin checks. It posts URL-encoded month-specific shortcode fields to
the public `eventon_get_events` operation. Request size is bounded by bootstrap
response limits, config field limits, and request/event caps. Redirects and
cookies are disabled; standard client request logging is disabled for this
named transport to avoid query disclosure. Existing restricted-header policy
applies identically in stored-schema tests and runtime. Authorization and stored
`X-WP-Nonce` are not forwarded by this public adapter.

The envelope must have `status: "GOOD"`, `json` occurrence inventory, and `html`.
Every occurrence `_ID` must uniquely match `event_` + `_ID` in the HTML cards;
counts and identities must agree. Authoritative `unix_start`/`unix_end` override
HTML dates and base-event dates, including repeats and DST. Stable IDs never
depend on array ordering. Identical cross-month occurrences deduplicate; conflicts
fail. Empty months do **not** stop traversal. `validation.minEventsPerFetch` applies
once across the complete window before freshness filtering, not per month.
For adapters, this is the only root validation option: event caps belong in
`adapter.options.maxEvents`, and required fields are enforced by the adapter,
not an ignored generic `requiredFields` list.

All-day Unix intervals are preserved exactly (EventON can publish inclusive
23:59:59 ends). A DST all-day interval may be 23 or 25 hours rather than 24.
The date-only HTML `startDate` marker identifies all-day presentation **only**;
its date/offset does not determine the authoritative instants. The internal
all-day flag propagates through ingestion snapshots and the existing published
event schedule flag without changing endpoint DTOs. Non-adapter paths retain
their existing default behavior. Inclusive end semantics are not silently changed.

The runtime bootstrap payload-hash shortcut is disabled for adapters, because
an unchanged shell says nothing about monthly responses, configuration changes,
or rolling-month progression. No internal retries are added: existing worker
retry policy reruns the whole read, refreshes config, and upserts stable occurrence
identities. Any month failure aborts before persistence.

No endpoint DTO changes are required. The schema contract is source configuration,
so validate it against `source-schema-v3.schema.json` before handoff. Both the
stored-schema test service and ingestion service call the same executor/adapter.
