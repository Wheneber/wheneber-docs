# Shared ingestion schedule policy

This policy applies to **all ingestion**, including preview/test-fetch and workers:
JSON (legacy/v3), HTML-lite/v3, RSS/Atom, ICS, and the Clark, EventON, Journey,
Ticketmaster, TicketWeb and Tockify adapters. It is not a title or source-ID rule.

| Publisher evidence after enrichment | Normalized schedule |
| --- | --- |
| Verified timed start and real end/duration | Preserve publisher schedule; no estimate disclaimer |
| Verified timed start, no end | Exactly **one elapsed hour**; `IsAllDay=false` |
| Verified event date, no start/end clocks | `IsAllDay=true`; local midnight to the next local midnight when no end is published |
| End clock only, invalid date/clock, conflicting schedule | Explicit failure or the parser's existing diagnosed skip; never rescue as all-day or estimate |

The generated timed-estimate disclosure is appended **once at the bottom**, after
detail enrichment and text sanitization:

> End time is estimated as one hour after the published start because the publisher provides no end time.

A missing description receives just that disclosure. Publisher descriptions are
otherwise retained. Authoritative enrichment removes the estimate flag and any
previous generated disclosure. All-day normalization does not add this timed-end
disclaimer. An explicitly published midnight timestamp is a clock, not date-only.

## Dates, zones and end boundaries

- A real event date is mandatory. Publication dates and the machine's current
  time are never implicit event dates. Explicit mapping is not proof that a
  publication date is an event date: source authors must map actual schedules.
- Date-only input is recognized **before** conversion to timestamps. Absence of
  an end is retained through materialization; synthesized timestamps are never
  re-inspected to guess provenance.
- Local midnight conversion uses the source timezone, not `startUtc + 24h`.
  Pacific 2026-03-08 is 23 hours and 2026-11-01 is 25 hours. Unknown configured
  timezones fail. Ambiguous/nonexistent local clocks fail for every format
  except ICS, whose RFC 5545 section 3.3.5 rule is applied instead: a repeated
  local time is its first occurrence and a skipped local time uses the UTC
  offset before the gap (see [ICS](ics.md)). Offset-bearing timed timestamps
  retain their absolute instant.
- Generic parsers use UTC when no source zone is supplied, as before. Configure
  the actual publisher zone for local/date-only schedules. ICS honors DTSTART/
  DTEND `TZID` and calendar `X-WR-TIMEZONE`; floating ICS uses that calendar
  zone, or UTC if absent. ICS DATE DTEND is **exclusive** and publisher duration
  is authoritative. Explicit date-only end fields in generic parsers are also
  exclusive when later than the start date; equal generic date-only bounds
  name one local calendar day, not a zero-hour event. ICS DTEND retains its
  RFC exclusive meaning without this generic date-range shorthand.
- Existing explicit all-day bounds are preserved, including EventON's inclusive
  Unix end and Tockify's publisher-provided zero/long durations. This change does
  not shift legitimate publisher ends.
- Ticketmaster `showDate` records require publisher timezone evidence for a
  date-only schedule; venue geography is not inferred as a zone. Unscheduled/
  cancelled/TBA records remain excluded.
- EventON repeat HTML dates are known unreliable: if the Unix end is absent but
  a potentially conflicting HTML end exists, extraction fails rather than
  overwriting that evidence with an estimate.
- Existing HTML yearless month/day anchoring remains unchanged and uses the
  source-local reference date to choose the year. A bare clock or missing
  month/day cannot become today.

`allDayWhenTimeMissing` remains accepted in existing schemas but no longer opts
individual sources into this behavior: date-only normalization is universal.
It cannot authorize invalid dates or turn an end-only clock into all-day.
Source `requiredFields` rules still apply; a schema requiring absent clocks
may reject a record before schedule construction. Update such a schema only
with source-specific evidence, not by manufacturing clocks.

## Reliability and operations

Identity is unchanged. Existing snapshot upsert/replay semantics apply; there
is no new retry loop, transport capability, arbitrary URL or authorization change.
Details are reconciled before final disclosure. Unsupported/ambiguous schedules
retain existing fail/skip diagnostics; Clark fails the whole source rather than
publishing a partial inventory.

Monitor source/stage/outcome/retry/failure-category/correlation telemetry.
Suggested thresholds (not provisioned alerts): any completeness failure, three
consecutive failures or >5 retries/source/15 minutes, any dead-letter growth
where a queue exists, freshness lag >2 poll intervals.

First response: avoid repeated source runs to contain bad output; isolate the
fetch/parse/map/validate/persist stage and occurrence IDs; compare publisher dates,
zones and end evidence; fix the responsible mapping/parser and rerun preview
before an authorized idempotent worker replay. Do not backfill or bulk-delete
records as an implicit repair.

The API and workers must both ship this behavior before hosted validation or
storing a schema dependent on it. Local validation does not establish deployed
support, and does not authorize submission or activation.
