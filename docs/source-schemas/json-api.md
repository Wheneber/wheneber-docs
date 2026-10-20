# JsonApi Source Guide

Use `JsonApi` when event data is delivered from a JSON endpoint.

All ingestion follows the [shared schedule policy](schedule-policy.md):
verified timed starts without ends receive one elapsed hour and a final
description disclaimer; verified dates without clocks are all-day in the
source timezone. Published ends take precedence. No event date is invented.

## Current Contract

JsonApi source submissions now use the v3 pipeline contract only.

- `schemaVersion` must be `3`
- `pipeline.calendar` and `pipeline.event` are required
- JSON list parsing config belongs in `pipeline.calendar.parser`
- request auth/headers/pagination belong in stage `fetch` nodes

For historical v1/v2 reference material, see [JsonApi V2 JSON Schema](json-api-v2-json-schema.md).

## When To Use JsonApi

Use `JsonApi` when:

- the website or widget fetches JSON event payloads
- event objects are in arrays or nested containers
- authenticated or paginated API requests are needed to fetch events

Do not use `JsonApi` when equivalent ICS or RSS feeds already provide the same data more simply.

## Minimum v3 JsonApi Shape

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Json",
      "parser": {
        "eventArrayPath": "$.events[*]",
        "mappings": {
          "title": "$.title",
          "startTime": "$.start"
        },
        "validation": {
          "requiredFields": ["title", "startTime"]
        }
      }
    },
    "event": {
      "type": "None",
      "input": { "mode": "none" },
      "parser": {}
    }
  }
}
```

## Common Optional Features

Add these only when needed:

- `pipeline.calendar.fetch.pagination` for page/offset/cursor APIs
- `pipeline.calendar.fetch.authentication` for bearer/api-key/basic auth
- `pipeline.calendar.fetch.requestHeaders` for required headers

### Anonymous JSON POST calendars

Generic v3 `Json` calendar stages support `fetch.method: "GET"` (the default)
or `"POST"`. Only POST may include `fetch.jsonBody`, a **JSON object**, not a
JSON-encoded string. The runtime sends UTF-8 `application/json; charset=utf-8`
when a body is present. POST without a body sends an empty POST; GET never sends
a body. Other methods, null/scalar/array bodies, and POST/body on non-JSON
calendars or code-owned adapters are rejected, not silently converted to GET.

```json
{
  "method": "POST",
  "jsonBody": {
    "categoryID": 0,
    "includePast": false,
    "filters": [],
    "search": ""
  },
  "pagination": {
    "strategy": "Page",
    "pageParamName": "page",
    "totalPagesPath": "$.pages"
  }
}
```

This is a `pipeline.calendar.fetch` fragment. The same method and literal body
are used for the initial request and each query-parameter pagination request;
page/offset/cursor values are **not** inserted into the body. Each attempt
creates a fresh request and content. POST endpoints must be read-only event
queries safe to repeat: workers may retry a failed run. No automatic POST retry
or write-operation semantics are introduced.

Bodies are bounded to 16384 bytes of compact UTF-8 JSON, depth 8 (root object at
depth 1), and 1024 nodes including containers and nulls. Size/depth/node limits
are runtime/admission checks, not fully expressible in JSON Schema.
Values are literal; date templates, bootstrap discovery, and body batching are
not supported. Do not submit permanently fixed dates as rolling coverage.

Community preview and submission accept safe anonymous JSON bodies but still
reject request headers, authentication, nested credential-like keys, and unsafe
embedded URLs. HTTPS/SSRF restrictions, no redirects/cookies, request/time and
response-size budgets remain unchanged. Body values never appear in fetch logs.
Embedded literal-IP and loopback URLs are conservatively rejected; body URLs
are data and are never dereferenced by the pipeline.
Event-detail enrichment and authentication-token bootstrap retain their existing
GET transport and policies; calendar bodies are never forwarded to them.
- `pipeline.calendar.parser.responseTransforms` for shaping split date/time fields

## Literal Defaults In Mappings

JsonApi mappings support `literal:` values for stable defaults.

Use this when source-level venue data is constant and per-event payloads are sparse.

```json
"mappings": {
  "title": "$.title",
  "startTime": "$.start",
  "venueName": "literal:The Museum",
  "venueAddress": "literal:1945 SE Water Ave, Portland, OR 97214",
  "location": "literal:1945 SE Water Ave, Portland, OR 97214"
}
```

Precedence note:

- event/detail stage value (if present)
- calendar-stage mapped value (including `literal:` defaults)
- empty/null

## Practical Pattern: Split Date And Time

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Json",
      "parser": {
        "eventArrayPath": "$.events[*]",
        "mappings": {
          "title": "$.title",
          "startTime": "$.startTime",
          "endTime": "$.endTime",
          "timeZone": "$.timeZone"
        },
        "responseTransforms": {
          "flattenPath": "$.events[*]",
          "fields": {
            "startTime": {
              "type": "Composite",
              "template": "{{startDate}} {{startClock}}",
              "inputs": {
                "startDate": "$.start.date",
                "startClock": "$.start.time"
              }
            }
          }
        }
      }
    },
    "event": {
      "type": "None",
      "input": { "mode": "none" },
      "parser": {}
    }
  }
}
```

## Validation Notes

- `schemaVersion` must be `3`
- `pipeline.calendar.type` should be `Json` for JsonApi sources
- `pipeline.calendar.parser.eventArrayPath` is required
- `pipeline.calendar.parser.mappings` should be non-empty
- `requestWorkflow` and pagination cannot both be configured in the same stage
- event links must be fully qualified `http`/`https` URLs
- relative links are resolved against `feedUrl`; non-URL values are treated as missing

## List Metadata + Detail Description Pattern

Some APIs provide complete list metadata (`title`, `dates`, `url`) but empty list descriptions.
In that case:

1. Keep calendar stage as `Json` for list extraction.
2. Set event stage to `Html` (or `Json`) and fetch per-event details from `eventUrl`.
3. Map `description` from detail content in `pipeline.event.parser.mappings`.
4. Validate that sample events now include non-empty descriptions.

## Recommended Workflow

1. Find the JSON endpoint returning real event objects.
2. Start from the minimum v3 shape.
3. Map `title` and `startTime` first.
4. Add URL, location, and optional enrichment fields.
5. Ensure mapped `url`/`eventUrl` values resolve to fully qualified `http`/`https` links.
6. Run `test-fetch` and inspect sample events.
7. Submit draft via community submissions.
