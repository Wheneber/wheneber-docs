# Journey Theater adapter

**Requires the Journey adapter backend deployment before storing this schema.**
Select code-owned `pipeline.calendar.adapter.name: "journey"`. Feed URL must be
`https://journeytheater.org/`. This is one logical source discovering publisher-linked
child calendars, not source-record creation or a general-purpose crawler.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "fetch": { "method": "GET" },
      "adapter": {
        "name": "journey",
        "options": {
          "futureMonths": 12,
          "maxRequests": 30,
          "maxFeeds": 10,
          "maxEvents": 1000
        }
      },
      "parser": {}
    },
    "event": { "type": "None", "input": { "mode": "none" }, "parser": {} }
  },
  "validation": { "minEventsPerFetch": 1 }
}
```

Options are required integers: `futureMonths` 1–24 (rolling months ahead),
`maxRequests` 4–50, `maxFeeds` 1–20, `maxEvents` 1–5000. Event count is additionally
server-capped. Optional root validation accepts only `minEventsPerFetch`. No
generic mappings, routing, auth, plugin names, arbitrary origins or ticket API.
Unrecognized options fail.

## Dynamic discovery and classification

Each execution rediscovers the official event-calendar iframe's Google calendar
`src` IDs, including base64 IDs. New embedded IDs need no schema change. The page's
public WordPress representation is used if its rendered path returns 403/404/503.
Do not infer one calendar per production: the publisher currently uses
program/area calendars.

Current homepage `.showcard-dates` cards provide the public production catalogue.
When root returns 403, the Journey adapter requests the public WordPress
`pages?slug=shows` representation through the shared bounded transport; the
adapter discovers its ID, follows all published child-page collection pages and
selects production templates. There are no fixed show IDs/titles/calendar IDs in
configuration or code. This fallback may lack a homepage-only advertised
production that has no published child page. The public-page catalogue is best
effort, not a guarantee of complete coverage.

Event summary must match one catalogue title (including an explicitly bounded
performance prefix/suffix or a longer name starting with a meaningful published
short title), and explicit season years constrain fallback matches.
Rehearsals, auditions, callbacks, classes/camps, training/tech, parties, office,
school-day and cue-to-cue entries are excluded. Unknown/ambiguous entries are
omitted with warning counts; a failed/empty discovery is an error, not success.
Unlinked calendars, arbitrary new layouts, unusual show names containing excluded
activity words, and publicly eligible showcases without clear catalogue matches
may require maintenance. Title matching is intentionally pragmatic, not AI-based.

## Schedules, details and missing venues

Ical.Net expands official calendar dates in a bounded window, preserving
RRULE/RDATE/EXDATE, recurrence overrides, cancellation and DST. The result is
independent occurrences, with no residual RRULE sent to the domain. IDs namespace
calendar identity + UID + original recurrence slot (RECURRENCE-ID for moved
instances), never array position. Independent calendars are not fuzzy-deduplicated.

Official calendar timestamps win. Ticket schedules are **not fetched or merged**.
Previously observed ticket/calendar conflicts therefore do not alter these
events; the adapter cannot detect future ticket changes it never fetches.
Publisher timezone for floating/date-only values is America/Los_Angeles.

Description and public URL come from the matching public show page/card, with
ICS description preferred. HTML is flattened to text and scripts/styles removed.
Optional description fetch failure does not discard an otherwise usable event.
Venue comes only from ICS LOCATION: no office address, ticket draft or guessed
shared venue. An absent venue uses the existing nonblank domain sentinel `TBD`,
with no fabricated venue/address/coordinates and an explicit missing-venue warning
count. The publisher skips geocoding that sentinel. It is **not a verified venue**;
events without coordinates may not appear in map/distance-based discovery.

## Reliability and security

All requests use the same bounded shared transport as test-fetch/runtime.
Only fixed HTTPS publisher page paths and validated Google public group-calendar
paths are fetchable; arbitrary iframe origins, ports, userinfo, redirects and
cookies are rejected/disabled. Authorization/Cookie/nonce headers are not forwarded.
Named transport decompresses bounded response streams; there is no browser,
external process, private-address URL input or arbitrary script execution.

Bootstrap hash-skip is disabled: unchanged page text does not imply unchanged
child feeds. A required calendar fetch/parse failure aborts before snapshots,
avoiding a smaller successful replacement inventory. Request/feed/event caps fail
explicitly. No internal retry loop: an authorized later retry repeats discovery
and upserts stable identities. Optional enrichment and classification omissions
are warning-counted and intentionally best effort.

Logs carry the existing SourceId/CorrelationId scopes and stage/outcome/retry
count/failure category plus discovered feeds, productions, emitted/excluded,
unmatched, missing venue and missing-description counts. Review warnings before
authorizing ingestion; schema-test success alone does not certify full coverage.
