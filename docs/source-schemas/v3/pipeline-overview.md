# Source Schema v3 Pipeline Overview

Source Schema v3 introduces a clean two-stage schemaDefinition contract:

- `calendar`: fetch and parse the source calendar/list payload
- `event`: enrich or transform each calendar item into final event fields

This model replaces implicit parser branching with explicit stage contracts.

## Goals

- Support flexible combinations for unexpected sources.
- Keep one canonical execution path for test-fetch and ingestion.
- Make parser composition explicit and statically validated.

## v3 shape

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Ics",
      "parser": {}
    },
    "event": {
      "type": "Html",
      "input": {
        "mode": "calendarFieldUrl",
        "field": "eventUrl"
      },
      "parser": {
        "mappings": {
          "description": ".event-description",
          "imageUrl": ".hero img@src"
        }
      },
      "limits": {
        "sameHostOnly": true,
        "maxRequestsPerRun": 5
      }
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

## Allowed calendar -> event combinations

- `Ics -> None`
- `Ics -> Html`
- `Ics -> Json`
- `Html -> Html`
- `Html -> Json`
- `Json -> Html`
- `Json -> Json`
- `Json -> None`
- `Rss -> None`
- `Rss -> Html`
- `Rss -> Json`

Combinations outside this matrix are rejected by schema validation.

## Stage contract summary

Calendar stage output must include enough fields for event stage input when event stage is not `None`.

Calendar stage fetch options may include:
- `fetch.authentication` for bearer, API key, or basic auth
- `fetch.requestHeaders` for non-restricted custom headers
- `fetch.pagination` for page, offset, or cursor traversal

For the backend v3 Html executor, detail occurrence expansion belongs in
`pipeline.event.parser`: set `expansion: "fanOut"`, `occurrenceSelector`, `mappings`,
and optional `fieldTransforms`. Relative mappings read the selected occurrence;
absolute XPath reads document-level metadata. The obsolete calendar `detailPage`
block is not the execution path. See [HTML detail extraction](html-detail-enrichment.md)
for identity, missing-schedule, minimum-count, and freshness semantics.

For Ics calendar parsers, calendar parser options can include `fieldTransforms` for deterministic per-field cleanup after RFC 5545 parsing and before event-stage processing.
This does not enable `mappings` for Ics.

Example Ics parser cleanup:

```json
{
  "pipeline": {
    "calendar": {
      "type": "Ics",
      "parser": {
        "fieldTransforms": {
          "location": [
            {
              "type": "regexReplace",
              "value": "^\\s*https://teams\\.microsoft\\.com[^,]*(?:,\\s*)?(.*)$",
              "replacement": "$1"
            },
            { "type": "trim" },
            { "type": "collapseWhitespace" }
          ]
        }
      }
    }
  }
}
```

If no cleanup is needed, keep `parser` as `{}`.

Event stage input modes:
- `none`
- `calendarFieldUrl`
- `fixedUrl`

Current implementation note:
- In current phase-2 scaffolding, `pipeline.event.input.field` is restricted to `eventUrl` when `event.type` is `Html` or `Json`.
- For `calendarFieldUrl`, you can optionally set `pipeline.event.input.baseUrl` to resolve/rebase detail links to the public content host.

## Event Input Modes Reference

The `pipeline.event.input.mode` determines how per-event details are obtained. Choose based on your source structure:

| Mode | Use Case | Example | Input Config |
|------|----------|---------|---------------|
| `none` | Events are already complete from calendar stage; no detail page needed | ICS feed with full event data | `{ "mode": "none" }` |
| `calendarFieldUrl` | Calendar stage outputs a URL field (e.g., `eventUrl`); fetch detail page per event | Calendar list has link to detail page in each item | `{ "mode": "calendarFieldUrl", "field": "eventUrl" }` |
| `fixedUrl` | All events use the same detail page URL pattern; substitute event ID into template | Detail URL is predictable: `https://example.com/events/{eventId}` | `{ "mode": "fixedUrl", "url": "https://example.com/events/{id}" }` |

### Decision Tree

For exact `calendarFieldUrl` constraints and `baseUrl` behavior, see [Event Input Constraints](event-input-constraints.md).

1. **Does your calendar stage output complete event data?**
  - Yes → Keep `event` lightweight, but still choose an allowed pair for your calendar type
  - For `Ics` and `Rss`, `event.type: "None"` is valid
  - For `Html`, choose `event.type: "Html"` or `"Json"` per the allowlist
  - No → Continue to step 2

2. **Does each calendar item have a link to its detail page?**
  - Yes → Use `mode: "calendarFieldUrl"`; in current phase-2 scaffolding map and use `eventUrl`
   - No → Continue to step 3

3. **Can you construct detail URLs from event IDs?**
  - Yes → Use `mode: "fixedUrl"`; provide URL template with `{id}` placeholder and validate behavior in your target environment
   - No → Check if detail page enrichment is needed; if not, use `mode: "none"`

### Examples

**Ics -> None (Complete calendar feed)**
```json
{
  "pipeline": {
    "calendar": { "type": "Ics" },
    "event": { "type": "None", "input": { "mode": "none" } }
  }
}
```

**Html -> Html with detail enrichment (Calendar has detail links)**
```json
{
  "pipeline": {
    "calendar": { "type": "Html", "parser": { "mappings": { "eventUrl": "a.event-link@href" } } },
    "event": { "type": "Html", "input": { "mode": "calendarFieldUrl", "field": "eventUrl" } }
  }
}
```

**Json -> Json with fixed template (Predictable detail URLs)**
```json
{
  "pipeline": {
    "calendar": { "type": "Json" },
    "event": { "type": "Json", "input": { "mode": "fixedUrl", "url": "https://api.example.com/events/{id}" } }
  }
}
```

## Security defaults

- `event.limits.sameHostOnly` defaults to `true`.
- `event.limits.maxRequestsPerRun` is required when event type is `Html` or `Json`.
- Event URL modes must use `http` or `https` only.

## Related files

- [v3 JSON Schema](source-schema-v3.schema.json)
- [v3 examples](pipeline-examples.md)

## Optional source adapter

TicketWeb Event Discovery WordPress calendars can select the closed `ticketweb`
adapter for public bootstrap/config discovery, monthly form POSTs, and JSON/popup
joins. See [TicketWeb adapter](ticketweb-adapter.md) for the default 12-calendar-month
window, required timezone/allowed origins, evidence-backed venue configuration,
show-versus-doors semantics and inferred ends.

EventON public calendars can select the closed `eventon` adapter in
`pipeline.calendar.adapter`. See [EventON source adapter](eventon-adapter.md)
for required explicit month windows, origin restrictions, authoritative
occurrence schedules, and whole-window completeness validation.

Journey Theater can select the closed `journey` adapter for dynamic
publisher-linked Google calendar discovery. See [Journey adapter](journey-adapter.md)
for best-effort public-show classification, official ICS recurrence semantics,
and the existing `TBD` missing-venue convention. This requires the corresponding
backend deployment; it is not supported by older schema executors.

### Adapter execution boundary

The backend resolves adapter names through an explicit dependency-injected
registry. The currently registered names are `eventon`, `journey` and `ticketweb`; unknown
names and duplicate registrations fail closed. Each adapter validates its own
typed options and owns its complete bootstrap, discovery, fallback, and parsing
lifecycle. Generic HTML, JSON, RSS, and ICS schemas continue through the normal
executor and do not receive adapter-specific behavior.

Both runtime ingestion and schema test-fetch use the same prepared adapter and
bounded fetch service. The service applies restricted-header filtering, response
size and timeout limits, safe host/path-only diagnostics, adapter-supplied URL
constraints, and the named adapter HTTP client with redirects and cookies
disabled. TicketWeb, EventON and Journey retain distinct URL scopes; only the EventON
public nonce is forwarded as the explicit nonce header. Test-fetch success does
not certify full source coverage; review adapter warnings and the source-specific
validation guidance before authorizing ingestion.
