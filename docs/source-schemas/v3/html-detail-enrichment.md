# HTML Detail Extraction and Occurrences

The backend v3 HTML executor uses `pipeline.event.type = "Html"` and
`pipeline.event.parser` for detail extraction. Put mappings and transforms there,
not in the obsolete `pipeline.calendar.parser.detailPage` block. Do not configure
both paths or supply placeholder dates to make unscheduled cards look valid.

## Single-event enrichment

Without `expansion: "fanOut"`, mappings run against the detail document and produce
one merged event per calendar seed. Non-empty detail values override calendar
values; missing/empty detail values retain calendar values. A failed detail fetch
may fall back to a genuinely scheduled calendar seed.

## Multiple occurrences

```json
{
  "type": "Html",
  "input": {
    "mode": "fixedUrl",
    "urlTemplate": "https://example.org{{eventUrl}}"
  },
  "limits": { "sameHostOnly": true, "maxRequestsPerRun": 10 },
  "parser": {
    "expansion": "fanOut",
    "occurrenceSelector": ".schedule",
    "mappings": {
      "startDate": ".date",
      "startTime": ".time",
      "timeZone": "literal:America/Los_Angeles",
      "description": "xpath=(//div[@class='description'])[1]",
      "location": "xpath=(//div[@class='venue'])[1]"
    },
    "fieldTransforms": {
      "startDate": [{ "type": "collapseWhitespace" }],
      "startTime": [
        { "type": "replaceLiteral", "value": " until ", "replacement": "-" },
        { "type": "trim" }
      ]
    }
  }
}
```

- `occurrenceSelector` is required for HTML fan-out and accepts CSS or node-selecting
  XPath (optionally prefixed with `xpath=`). No matching nodes means no occurrences.
- Mappings run relative to each selected occurrence. Use `.` for its text,
  `@attribute` for its attribute, CSS such as `a@href`, or relative XPath.
- Absolute XPath (`//...` or `/...`) reads document-level metadata even while
  processing an occurrence. Parenthesized XPath such as `(//div[@class='venue'])[1]`
  selects the first document-level match. XPath must select nodes, not scalar
  `string(...)` / `normalize-space(...)` results.
- `fieldTransforms` has the same ordered operations as HTML calendar mappings:
  `trim`, `collapseWhitespace`, `replaceLiteral`, and `regexReplace`. Replacement
  operations use `value`, `replacement`, and optional `ignoreCase`. Regex uses .NET
  syntax (`$1` for capture replacements) and a bounded timeout. Unknown transforms
  fail extraction rather than silently falling back.
- HTML entities are decoded before transforms.
- Map `startDate`, `startTime`, and `timeZone` for local times. Time ranges can share
  a trailing meridiem (`3:00-6:00 PM`) or specify both (`9:00am to 4:00pm`).
  A missing/unparseable schedule is skipped; dates are never synthesized. A local
  date must include a year; bare clock values cannot become "today" via date parsing.
- Fan-out IDs use the merged `id` (falling back to detail URL) plus `::` and the
  UTC start timestamp in round-trip format. Reordering nodes does not change IDs.
  Identical identities at the same start within a seed are deduplicated. A changed
  start creates a new identity; distinct simultaneous events need distinct mapped IDs.
- A failed/blocked detail fetch in fan-out produces no seed fallback occurrence.
  Request limits count attempted requests, including failures. There is no immediate
  detail retry; later source runs can fetch again.
- `maxEventsPerFetch` limits returned occurrences, not just calendar cards.

Existing JSON `eventArrayPath` fan-out and index-based JSON identities are unchanged.

## Source coverage and location quality

Validate every listing card against its current detail page, not a fixed expected
event count from an older snapshot. One annual event can have multiple simultaneous
sessions at different venues. Map a distinct `id` for each such session (for example,
its explicit park name) so the shared start timestamp does not collapse them.
Map a shared page date with absolute XPath and per-session time/location with
relative XPath when that matches the source structure.

Do not use a navigation/sidebar contact address as proof of an event's venue.
Prefer the venue stated in the event body. If a specific venue is not published,
a publisher-supported city-level calendar `location` default may be retained by
the normal non-empty merge policy. Document that this is city-only and do not
populate a fabricated specific venue/address. Retrospective results without a
next date/time schedule must not create new occurrences.

## Validation and freshness

Root `validation.minEventsPerFetch` is enforced by stored-schema tests and runtime
ingestion. It counts successfully parsed occurrences with non-empty title and
location **before** the freshness filter, within configured request/event limits.
It does not count listing cards or require a minimum number of future events.
Past-only calendars can therefore pass and return zero upcoming events.

Failure diagnostics include `required`, `parsed`, `validBeforeFreshnessFilter`, and
`upcomingAfterFilter`. Runtime fails before snapshot writes or publication. An
omitted minimum preserves empty-result success; `0` explicitly permits it.
The compact test response count/sample remains freshness-filtered (end time within
the past 24 hours or later). Root `requiredFields` is not an arbitrary field-rule
engine in this executor; the core title/location policy is fixed.

HTML detail sources are re-extracted even when the listing payload hash is
unchanged, since detail schedules can change independently. Configured minimums
are also rechecked on unchanged payloads. Inspect representative descriptions,
locations, source URLs, and both ends of each schedule, not just card counts.
