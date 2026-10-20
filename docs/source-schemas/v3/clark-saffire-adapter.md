# Clark County Saffire adapter

`clark-saffire` is a **Clark-only**, code-owned adapter, not a general Saffire
integration or a workflow scripting language. The source type is `HtmlLite`;
`feedUrl` must be exactly `https://www.clarkcoeventcenter.com/events`.

## Contract

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "fetch": { "method": "GET" },
      "adapter": {
        "name": "clark-saffire",
        "options": {
          "maxDates": 200,
          "maxRequests": 100,
          "maxEvents": 100,
          "maxElapsedSeconds": 180,
          "maxTotalResponseBytes": 20000000
        }
      },
      "parser": {}
    },
    "event": { "type": "None", "input": { "mode": "none" }, "parser": {} }
  },
  "validation": { "minEventsPerFetch": 1 }
}
```

All five options are required. Bounds: dates 1–200, requests 3–100,
event-day records 1–100, elapsed seconds 5–180, aggregate decoded response
bytes 8,192–20,000,000. Each response is independently capped at 2,000,000
bytes (or a lower transport configuration limit). The effective runtime/
preview event cap can be lower; overflow fails rather than truncating.

No custom headers (even an empty header block), authentication, bodies,
origins, URLs, selectors, routing, enrichment stages or extra configuration
are allowed. This is the **only** adapter allowlisted for community
test-fetch/submission; other adapters remain unavailable there. Anonymous
pre-production submission/approval semantics are unchanged.

## Discovery, transport and identity

1. Fetch the HTML listing. Extract exactly one inline
   `var eventDays = new Array("MM/dd/yyyy",...)` bootstrap as a JSON string
   list, **never executing JavaScript**. Invalid, duplicate, unordered or
   excessive dates fail. Dates may span years; none are hard-coded.
2. Preserve publisher ordering and POST batches of at most five dates to
   `/services/eventsservice.asmx/GetEventDaysByList`. Adapter code owns the
   exact public request shape, including string `"false"` filter flags and
   boolean `searchInDescription`. The existing shared `method`/`jsonBody`
   transport sends `application/json; charset=utf-8`.
3. Require object-valued `d` and consume `d.Days[].Times[].Items[]`.
   Every requested date must appear exactly once with at least one item.
   `DateString` is a calendar bucket; the ASMX epoch date is **not** a
   Pacific clock.
4. Fetch each distinct same-origin `/events/{year}/{slug}` detail once per
   execution. There is no persistent detail cache or stale-on-error reuse.
5. Identity is `clark_{EventID}_{YYYY-MM-DD}_{sorted-EventItemIDs}`.
   Identical repeated identities deduplicate; conflicting duplicates fail.
   Multiple conflicting sessions without distinct publisher item identities
   are unsupported, not collapsed to one event.

Preview and worker always use the DNS-guarded public transport, HTTPS,
fixed origin/path allowlist, no redirects, no proxy, no cookie storage and
no credentials. A linked deadline bounds the entire execution, including
body reads, and caller cancellation propagates.

## Authoritative schedules and completeness

The [global ingestion schedule policy](../schedule-policy.md) applies, not a
Cinema/PDXLAN title or publisher-ID exception. The same policy runs in preview
and worker execution.

The detail page must supply consistent `meta[itemprop=startDate/endDate]`
and one `itemprop=description` region. Month/day prose is anchored only to
these publisher years, never the current year or URL year. A date outside
the metadata range fails.

- Date-specific daily prose takes precedence over shared ASMX hours,
  including different closing hours on different days. Overrides are logged.
- Explicit `Hours:`/`Times:` ranges can apply across the metadata interval.
  Otherwise an explicitly specified, internally consistent service range
  supplies the clocks.
- Published clocks need explicit meridiem. Abbreviated `3-8pm` or
  `4:00 - 6:00pm` is accepted **only** when both clocks independently agree
  with the fully explicit public service range and its numeric start.
- An explicit start with no end receives one elapsed hour and the shared
  estimated-end disclaimer once, after enrichment. Actual published ends win.
  Approximate durations are not treated as authoritative ends.
- Date-labelled program titles without clocks do not override a shared
  showtime. Actual daily clock overrides still take precedence.
- Explicit all-day and verified event dates without any published clocks use
  Pacific local midnight to **next local midnight (exclusive)**, including
  23/25-hour days. Each inventory day remains a separate occurrence; the whole
  multi-day metadata interval is not duplicated on every day.
- Doors/early entry do not establish performance starts. Unsupported clock
  evidence, overnight ranges, DST gaps/folds, ambiguous/multiple daily schedules,
  unmatched daily clock labels and unknown formats still fail explicitly.

Any unresolved occurrence fails the **entire source**: no partial success,
sample, snapshot writes or publication. Diagnostics identify occurrence and
category without including response bodies, headers or session identifiers.
Community preview exposes the adapter's bounded safe schedule diagnostics;
other public errors retain the generic error response.

## Current source readiness

Anonymous live verification on **2026-10-19** with the user-approved global
policy confirmed **40 dates, 64 event-day records, 27 event IDs**, including 20
structured-time records. Worker and preview each validated **64/64**, with zero
unresolved records. Nine Cinema occurrences use the disclosed one-hour estimate;
four clockless PDXLAN dates are all-day. These are observations, not policy IDs
or a fixed future coverage expectation.

All 64 stable occurrence IDs were unique, all local dates matched inventory,
and there were no overlapping daily duplicates. All 64 mocked worker snapshots
matched mapped output; all ten preview samples matched core/detail fields.
Replay added zero new snapshots and updated the same 64.

The package is **locally ready for hosted validation after deployment**.
Hosted support has not been verified. Both API and worker require deployment
of the adapter/allowlist and global policy before hosted preview; **no Draft,
submission, activation, deployment or backfill was performed**.

## First response and reliability

No internal retry loop: transport failures fail immediately. A worker retry
restarts discovery and all public reads. Stable publisher identities make
successful replay idempotent through existing snapshot upsert behavior.
Failure before adapter completion produces a failed run and no snapshots.

Use source identifier, stage, outcome, retry count, failure category and
correlation identifier to isolate `discovery`, `inventory`, or
`daily_schedule`. Suggested alert thresholds for future operation:

- any completeness/parse failure (expected complete success rate 100%);
- three consecutive failures or more than five retries/source/15 minutes;
- any dead-letter growth if a dead-letter queue is introduced (this adapter
  has no queue or replay endpoint);
- freshness lag greater than two configured poll intervals.

Contain repeated work by leaving the source unactivated; inspect the
specific stage and safe occurrence IDs, compare current public schedules,
fix fixtures/configuration if warranted, and rerun preview followed by one
normal ingestion cycle. Do not omit problematic events to recover.
