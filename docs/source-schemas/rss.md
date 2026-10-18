# Rss Source Guide

Use `Rss` when the source publishes RSS or Atom and each item behaves like an event record.

For a machine-readable authoring contract, use the contributor-facing [Rss JSON Schema](rss.schema.json).

## When To Use It

Use `Rss` when:

- the response is XML with `<rss>`, `<channel>`, `<item>`, or Atom `<entry>` nodes
- titles and dates can be extracted from feed elements
- the feed is event-oriented rather than general news content

Do not use `Rss` when the feed is missing event dates or when a better ICS feed already exists.

## Required `schemaDefinition` Shape

`Rss` uses v3 pipeline. The RSS-specific contract applies to `pipeline.calendar.parser` and requires `mappings`.

```json
{
  "mappings": {
    "title": "title",
    "description": "description",
    "startTime": "pubDate",
    "url": "link",
    "id": "guid",
    "location": "location"
  },
  "filters": {
    "includeAny": [
      { "field": "title", "value": "showcase", "matchType": "contains" }
    ],
    "excludeAny": [
      { "field": "description", "value": "recap", "matchType": "contains" }
    ]
  },
  "validation": {
    "minEventsPerFetch": 1,
    "requiredFields": ["title", "startTime"]
  }
}
```

## How To Structure The Payload

- `type` must be `Rss`
- `feedUrl` points at the XML feed
- `schemaDefinition.pipeline.calendar.parser.mappings` maps output fields to XML elements or attributes per item
- `validation.minEventsPerFetch` should default to `1`
- `validation.requiredFields` should default to `title` and `startTime`

## Mapping Values

Each mapping value is resolved against one `<item>` (RSS) or `<entry>` (Atom):

- `title` — first child element by local name (works for RSS 2.0 and Atom)
- `dc:creator` — prefixed child element, resolved through the feed's namespace declarations
- `enclosure@url` — attribute of a child element; `@attr` reads an attribute of the item itself
- An Atom `<link href="..."/>` with no text yields its `href`

Mapping keys `guid`, `link`, and `timezone` are aliases for `id`, `url`, and `timeZone`.
Element text is used as published; HTML inside `description` is not converted to a schedule.

## Schedules

- The runtime never invents a schedule. There is no implicit `pubDate`, current-time,
  placeholder-title, or placeholder-location fallback.
- `pubDate` is article metadata. Map it as `startTime` only when the publisher really
  uses it as the event start.
- Items whose mapped schedule is missing or does not parse as an absolute date/time
  (with an explicit date, or an ISO/RFC timestamp) are skipped and counted in a
  `missing_or_invalid_schedule` warning. `validation.minEventsPerFetch` turns a feed where
  every item is skipped into a failed run/test.
- A mapped `endTime` is preserved. Without one, the shared two-hour default duration applies.

## Detail Enrichment (`Rss -> Html` / `Rss -> Json`)

When the feed lacks schedule or venue data, keep `pipeline.calendar.type: "Rss"` and add an
event stage, for example `pipeline.event.type: "Html"` with
`input: { "mode": "calendarFieldUrl", "field": "url" }` and detail `parser.mappings`
(`startTime`, `endTime`, `location`, `venueName`, `venueAddress`, `description`, `url`,
`imageUrl`) plus optional `fieldTransforms`. Each RSS item is a seed; non-empty detail
fields override seed fields. Detail requests honor `limits.sameHostOnly` and
`limits.maxRequestsPerRun`. If a detail fetch fails or yields no valid schedule, the item
is skipped unless its own RSS mappings supply a valid `startTime`.

Runtime ingestion and `test-fetch` share this execution path, request headers
(`pipeline.calendar.fetch.requestHeaders`), and field projection.

## Requirements And Validation Notes

- `mappings` is required; a schema without mappings is rejected
- `startTime` must be mapped when `pipeline.event.type` is `None`
- each extraction rule value should resolve against each feed item
- `startTime` must parse as a real date/time
- `id` is recommended for stable identity (map from `guid` when available)
- `url` should point to the event detail page when available (map from `link`)
- `parser.filters` and `parser.eventEnrichment` are accepted by the JSON Schema but are not
  executed by the current runtime; use a narrower feed URL and `pipeline.event` instead
- `location` or `endTime` should be added to `validation.requiredFields` only when the feed is consistently complete and dropping missing items is intentional

## Filtering Expectations

- Prefer an event-only or category-specific upstream feed URL when the publisher offers one.
- If upstream filtering is not sufficient, use `pipeline.calendar.parser.filters` with deterministic include/exclude rules (schema-valid, but not yet executed by the runtime — verify sample output).
- Keep filter rules explicit and reviewable (`field`, `value`, optional `matchType`, optional `caseSensitive`).
- If the feed mixes event items with blog/news/announcement content and cannot be narrowed upstream or with deterministic parser filters, choose a different source URL or source type.
- If you enable detail enrichment, make sure `url` (or `link`) is mapped because the detail stage depends on it even though it is not part of `requiredFields`.

## Recommended Contributor Workflow

1. Inspect the feed structure and confirm that each item is an event.
2. If the publisher offers multiple RSS endpoints, pick the narrowest event-oriented feed before authoring the schema.
3. Map `title` and `startTime` first.
4. Add `validation.minEventsPerFetch = 1` and `validation.requiredFields = ["title", "startTime"]`.
5. Add `description`, `url`, `id`, and `location`.
6. Run `test-fetch` and verify that the sample items parse correctly.
7. Tighten `requiredFields` only if the feed is consistently complete and you intentionally want incomplete items dropped.
8. Add deterministic `filters.includeAny`/`filters.excludeAny` rules if non-event items remain after source-side narrowing.
9. Submit the draft with `community-submissions`.

## Common Failure Modes

- using a general-purpose blog/news feed instead of an event feed
- `startTime` points to a non-date field
- missing `id` mapping leads to unstable identity
- element names differ between RSS and Atom variants
- using broad keyword filters without deterministic review criteria
- adding `location` or `endTime` to `requiredFields` too early and filtering out otherwise valid events

See also: [Submission API And Validation](submission-api-and-validation.md)