# Submission API And Validation

This page describes the shared contributor workflow for every source type.

> **Pre-production temporary access:** anonymous preview, submission, submission retrieval, admin-ingestion review/listing/mutation/run, and source reactivation are intentionally allowed. This is an explicit pre-production risk acceptance, not a production security design. Do not deploy with this access model. The production-blocking checklist is tracked in the repository root `TODOs.md`.

- `POST /api/source-schemas/test-fetch` — non-persisting public-feed preview.
- `POST /api/source-schemas/community-submissions` — independently revalidates and stores a Draft.
- `GET /api/source-schemas/community-submissions/{sourceSchemaId}` — signed-in users can retrieve only their own submission; anonymous requests can retrieve any anonymous submission (shared pre-production semantics).

Preview and submission do not require an access token. Anonymous submissions are attributed truthfully in storage as `SubmittedBy = "anonymous"`; no user or tenant identity is fabricated. Signed-in submissions continue to store Entra tenant/object attribution and retain owner-only retrieval. Anonymous submissions are intentionally shared in this pre-production environment. Anonymous idempotency is scoped by the server-observed remote IP, key, and request fingerprint: an exact retry from the same network scope replays the same submission, while different payloads or remote IPs receive distinct deterministic IDs. The IP is used only as an idempotency namespace and is not persisted as submitter attribution. Signed-in submissions retain same-key/different-payload `409` behavior. All submissions require an `Idempotency-Key` header (8–128 ASCII letters, digits, hyphens, or underscores).

Successful submission returns `201 Created`, a `Location` pointing to the retrieval resource, the persisted source details, and validation results. The persisted status is always `Draft`; creation never approves or activates a source. During pre-production, the controller-wide AdminOnly policy is disabled. Review/detail, listing, schema/feed URL updates, ingestion validation/runs, and reactivation are public. A reviewer must inspect the source details before calling reactivation; this is a process requirement only and is not enforced by an authorization or approval workflow. A few existing maintenance actions retain their narrower action-level AdminOnly policies. Before production, restore Entra authentication, AdminOnly access for all admin routes including review and approval, and verified contributor attribution/ownership.

Contributor fetches require a public HTTPS endpoint. Private, loopback, link-local, and metadata IP ranges are blocked; DNS results are checked at connection time; redirects are not followed; follow-on requests must remain on the feed origin; and requests are bounded by a 15-second per-request timeout, 45-second total validation timeout, 2 MB response limit, 100 total outbound requests, a 100-event preview cap, and a 2 MiB HTTP request body. The 100-request budget supports feeds with many detail-page lookups while keeping validation bounded. Authentication, request headers, and credential-bearing URLs are not accepted. A rate limit of five requests per minute applies per signed-in object ID (or remote IP for anonymous calls), so anonymous callers behind the same IP share a bucket. This limit is not a production abuse control. A candidate must yield at least one upcoming event with a usable title, schedule, and location before a Draft is stored.

## v3 Pipeline Contract (Current)

Wheneber uses the `schemaDefinition` v3 contract for new source authoring and submission, with explicit calendar and event parser stages.

- Overview: [Pipeline V3 Overview](v3/pipeline-overview.md)
- Examples: [Pipeline V3 Examples](v3/pipeline-examples.md)
- JSON Schema: [Pipeline V3 JSON Schema](v3/source-schema-v3.schema.json)

The v3 model is the active clean-break contract for contributor-facing payloads.

Additional v3 references:

- Migration: [Pipeline V3 Migration Guide](v3/migration-guide.md)
- Allowed calendar/event pairs: [Pipeline V3 Type Pair Reference](v3/type-pair-reference.md)
- Event input constraints: [Pipeline V3 Event Input Constraints](v3/event-input-constraints.md)
- Html enrichment patterns: [Html Detail Enrichment Patterns](v3/html-detail-enrichment.md)

Do not use admin-only approval or trigger endpoints as a community contributor.

## Shared Request Envelope

Both `test-fetch` and `community-submissions` accept the same payload structure. Build once, test once, submit once:

**Machine-readable schema:** See [submission-request.schema.json](submission-request.schema.json) for the complete JSON Schema definition. Use this to validate or generate payloads programmatically.

```json
{
  "name": "Example Community Center Events",
  "description": "What this source publishes",
  "type": "JsonApi",
  "feedUrl": "https://example.org/events",
  "schemaDefinition": "{\"schemaVersion\":3,\"pipeline\":{\"calendar\":{\"type\":\"Json\",\"parser\":{\"eventArrayPath\":\"$.events[*]\",\"mappings\":{\"title\":\"$.title\",\"startTime\":\"$.start\"}}},\"event\":{\"type\":\"None\",\"input\":{\"mode\":\"none\"},\"parser\":{}}}}",
  "metadata": {
    "location": "Example City",
    "category": "community",
    "region": "WA",
    "language": "en",
    "estimatedEventCount": 20,
    "eventStartNotBeforeDate": "2026-01-01",
    "eventFallbackUrl": "https://example.org/events",
    "customAttributes": {
      "defaultLocation": "123 Main St, Example City, WA 99999"
    }
  },
  "calendarStrategyOverride": "Auto"
}
```

**Field requirements:**

- `type` must be one of `Ics`, `Rss`, `JsonApi`, `HtmlLite`
- `feedUrl` is required and must be an absolute public HTTPS URL without credentials or fragments
- **`schemaDefinition` must be sent as a JSON string, not a nested object** — wrap the v3 JSON body in quotes and escape its inner quotes.
- `schemaDefinition` must use v3 shape with `schemaVersion: 3` and `pipeline`
- `schemaDefinition` can be up to 200000 characters
- `type` must match `pipeline.calendar.type` (`JsonApi` corresponds to calendar type `Json`)
- only `calendarStrategyOverride: "Auto"` is accepted
- `url` and `eventMapping` are not valid top-level submission fields
- `name` and `description` are required for submission; `metadata` is recommended
- **`name` must identify the organization and event collection in plain language** — use `[Organization] Events` or `[Organization] [Collection]` (for example, "Battle Ground Parks & Recreation Events" or "Portland Art Museum Calendar"); do not include source type identifiers such as "ICS," "RSS," "HtmlLite," "JsonApi," or the word "Feed" — those are platform internals invisible to users

Current v3 guardrails to account for during authoring:

- `pipeline.event.input.field` currently expects `eventUrl` for phase-2 scaffolding when `event.type` is `Html` or `Json`
- Some type pairs are rejected (for example `Html -> None`); use the allowlist in [Pipeline V3 Type Pair Reference](v3/type-pair-reference.md)

JsonApi event-link clarification:

- Map `url` as the canonical event link key in `pipeline.calendar.parser.mappings` when the source exposes per-event detail links.
- Runtime output and `sampleEvents` may still show `eventUrl` as a normalized alias, depending on parser behavior. That alias does not change the authored mapping key.

**Metadata fields** (all optional, but recommended):

| Field | Type | Example |
|-------|------|---------|
| `location` | string | `"Seattle, WA"` or `"Pike Place Market"` |
| `region` | string | `"WA"` or `"Pacific Northwest"` |
| `category` | string | `"community"`, `"government"`, `"entertainment"`, or similar |
| `language` | string | `"en"`, `"es"`, `"fr"` (ISO 639-1 codes) |
| `contactEmail` | string | `"admin@example.org"` for questions about this source |
| `estimatedEventCount` | integer | `50` (approximately how many events per month) |
| `eventStartNotBeforeDate` | string (`YYYY-MM-DD`) | `"2026-01-01"` to ignore older events |
| `eventFallbackUrl` | string (absolute URL) | `"https://example.org/events"` used only when an event has no URL |
| `customAttributes.defaultLocation` | string | default event `location` fallback |
| `customAttributes.defaultVenueAddress` | string | default event `venueAddress` fallback |

Metadata fallback/date policies are applied consistently in preview and ingestion. `eventStartNotBeforeDate` compares the event's UTC start date (the source parser's normalized instant), `eventFallbackUrl` fills a missing source URL, and the two `customAttributes` fallbacks only fill their corresponding missing fields. Only `calendarStrategyOverride: "Auto"` is supported. Unknown top-level and metadata keys are rejected rather than ignored.

Metadata notes:

- `eventStartNotBeforeDate` is a policy filter for both `test-fetch` and runtime ingestion.
- `eventFallbackUrl` is applied only when an event URL is missing.
- `customAttributes.defaultLocation` and `customAttributes.defaultVenueAddress` are independent and do not cross-fill.
- Avoid ad-hoc metadata keys outside the fields listed above.

`schemaDefinition` must contain only fields supported by the selected source type. Do not include wrapper or note fields such as:

- `sourceName`
- `baseUrl`
- top-level `transforms`

Those do not belong in the submitted schema body.

Clarification:

- top-level `baseUrl` is invalid
- `pipeline.event.input.baseUrl` is valid in v3 when using `mode: "calendarFieldUrl"`

## Wrong Vs Right: ICS Request Envelope

This is a common cross-contract mistake when using examples from other systems.

**Common mistake** (from other calendar APIs):

```json
{
  "type": "ics",
  "url": "https://calendar.google.com/calendar/ical/lacentercalendar%40gmail.com/public/basic.ics",
  "eventMapping": {
    "id": "uid",
    "title": "summary",
    "description": "description",
    "location": "location",
    "startTime": "dtstart",
    "endTime": "dtend",
    "url": "url"
  }
}
```

**Correct payload** (works for both test-fetch and community-submissions):

```json
{
  "name": "La Center Calendar",
  "description": "Public ICS feed from La Center, WA",
  "type": "Ics",
  "feedUrl": "https://calendar.google.com/calendar/ical/lacentercalendar%40gmail.com/public/basic.ics",
  "schemaDefinition": "{}",
  "metadata": {
    "location": "La Center",
    "category": "community",
    "region": "WA"
  }
}
```

For `Ics`, `schemaDefinition` is commonly `{}` for baseline parsing, but may include `validation`, `eventEnrichment`, and optional `fieldTransforms` for deterministic cleanup.

For `Ics`, `fieldTransforms` runs after RFC 5545 extraction and before validation checks. It is per-field cleanup only and does not enable cross-field mapping.

## Reading `test-fetch` Results Carefully

Treat `sampleEvents` as diagnostic examples, not as a guaranteed sorted list of the newest events.

- `sampleEvents` may include older events even when the source feed still contains current or future events.
- If an ICS feed looks unexpectedly stale based on `sampleEvents`, inspect the raw feed before concluding the source is unusable.
- Compare `totalEventsParsed` with the raw feed when counts look suspiciously low, especially for recurring calendars or long-running Google Calendars.
- For Source Creation Specialist validation, use the hosted `https://dev.api.wheneber.com/api/source-schemas/test-fetch` path so results reflect the canonical contributor workflow.
- For HtmlLite and Wix sources that use detail enrichment, judge success by sample quality and plausible field coverage first; do not require exact count parity with the live page during hosted validation.

Policy parity reminder:

- Treat `test-fetch` as a policy preview for ingestion.
- If you use metadata policies such as `eventStartNotBeforeDate`, `eventFallbackUrl`, or fallback custom attributes, validate expected behavior in `sampleEvents` before submission.

Practical ICS check:

1. Fetch the raw `.ics` response.
2. Confirm it contains current or future `VEVENT` entries.
3. If the raw feed is current but `sampleEvents` look old or `totalEventsParsed` seems unexpectedly low, treat that as a validation/troubleshooting signal rather than immediate evidence that the source itself is stale.

---

## Common Mistakes (And How to Avoid Them)

### ❌ Mistake 1: `schemaDefinition` as an object instead of a string

**Wrong:**
```json
{
  "schemaDefinition": {}
}
```

**Right:**
```json
{
  "schemaDefinition": "{}"
}
```

**Why:** The backend expects a JSON string. If you're using PowerShell or JavaScript, wrap your JSON in quotes or use `.ConvertTo-Json -Depth 5`.

---

### ❌ Mistake 2: Custom metadata fields

**Wrong:**
```json
{
  "metadata": {
    "source": "ridgefieldwa.us",
    "city": "Ridgefield",
    "state": "WA",
    "country": "USA",
    "format": "iCalendar"
  }
}
```

**Right:**
```json
{
  "metadata": {
    "location": "Ridgefield, WA",
    "region": "WA",
    "category": "government",
    "language": "en",
    "eventStartNotBeforeDate": "2026-01-01",
    "eventFallbackUrl": "https://ridgefieldwa.us/events",
    "customAttributes": {
      "defaultLocation": "510 Pioneer St, Ridgefield, WA 98642"
    }
  }
}
```

**Why:** The API only recognizes specific metadata fields: `location`, `region`, `category`, `language`, `contactEmail`, `estimatedEventCount`, `eventStartNotBeforeDate`, `eventFallbackUrl`, and `customAttributes`. Unknown keys are rejected. Descriptive fields are retained on the source schema; fallback and date policies are applied by preview and ingestion.

Allowed `customAttributes` keys for fallback behavior:

- `defaultLocation`
- `defaultVenueAddress`

These two keys are independent and intentionally do not cross-fill each other.

---

### ❌ Mistake 3: Submitting before testing

Always test first with `/api/source-schemas/test-fetch`. If test returns `totalEventsParsed: 0`, debug before submitting. A submission that fails fetch or schema validation returns `422` and does not persist a draft.

When using metadata fallback/date policies, verify all of the following in test-fetch first:

- old events are filtered as expected by `eventStartNotBeforeDate`
- missing event URLs are populated by `eventFallbackUrl`
- location fallback behavior matches your intended `defaultLocation` and `defaultVenueAddress`

### ❌ Mistake 4: Using unsupported event input field

**Wrong (current phase-2 scaffolding):**
```json
{
  "pipeline": {
    "event": {
      "type": "Html",
      "input": { "mode": "calendarFieldUrl", "field": "url" }
    }
  }
}
```

**Right:**
```json
{
  "pipeline": {
    "calendar": {
      "parser": {
        "mappings": { "eventUrl": "a@href" }
      }
    },
    "event": {
      "type": "Html",
      "input": { "mode": "calendarFieldUrl", "field": "eventUrl" }
    }
  }
}
```

---

### ❌ Mistake 5: Expecting list payload `description` to be complete

Some Json list APIs return rich list metadata but blank description fields.
If `sampleEvents.description` is empty after list parsing:

1. Add event-stage enrichment (`pipeline.event.type = "Html"` or `"Json"`).
2. Use `input.mode: "calendarFieldUrl"` with `field: "eventUrl"`.
3. Set `pipeline.event.input.baseUrl` when list links are relative or host-mismatched.
4. Map `description` from detail content in `pipeline.event.parser.mappings`.
5. Re-run test-fetch and verify description is populated in sample events.

---

## Contributor-Safe Workflow

### Environment: Choose Your Endpoint

For the Source Creation Specialist and other hosted validation workflows, use the hosted app base below. Do not switch this workflow to localhost.

| Environment | Endpoint Base |
|-------------|---------------|
| **Hosted validation / Source Creation Specialist** | `https://dev.api.wheneber.com/api/source-schemas` |

⚠️ **Use `https://dev.api.wheneber.com/api/source-schemas` for test-fetch and contributor draft submission in this workflow. Do not use localhost for Source Creation Specialist validation.**

### 1. Load A Starter Template

Use the type guides and v3 examples linked above as the known-good starting points. There is no templates API endpoint.

### 2. Test Fetch Before Submitting

Use `POST /api/source-schemas/test-fetch` while refining the schema. Send the complete payload you plan to submit:

```json
{
  "name": "Example Recreation Events",
  "description": "Community events from Example Org",
  "type": "Rss",
  "feedUrl": "https://example.org/events/feed.rss",
  "schemaDefinition": "{\"schemaVersion\":3,\"pipeline\":{\"calendar\":{\"type\":\"Rss\",\"parser\":{\"mappings\":{\"title\":\"title\",\"startTime\":\"pubDate\",\"eventUrl\":\"link\",\"id\":\"guid\"},\"filters\":{\"includeAny\":[{\"field\":\"title\",\"value\":\"event\",\"matchType\":\"contains\"}],\"excludeAny\":[{\"field\":\"description\",\"value\":\"recap\",\"matchType\":\"contains\"}]}}},\"event\":{\"type\":\"None\",\"input\":{\"mode\":\"none\"},\"parser\":{}}},\"validation\":{\"minEventsPerFetch\":1,\"requiredFields\":[\"title\",\"startTime\"]}}",
  "metadata": {
    "location": "Example City",
    "region": "WA",
    "category": "community"
  }
}
```

Test-fetch validates the schema parsing and returns event samples. If test succeeds with good event count, proceed to submission with the same payload.

Important note for hosted validation:

- use the hosted app endpoint for validation so your test-fetch path matches the intended contributor workflow
- when counts look lower than the raw source, compare hosted sample events against the page/feed before rewriting working selectors
- do not move the same payload to localhost just to chase count differences

Additional troubleshooting:

- if the hosted endpoint is unavailable, pause the workflow and record the outage rather than swapping to a local backend instance

### 3. Submit The Draft

Use `POST /api/source-schemas/community-submissions` once test-fetch is stable.

The response returns:
- `sourceSchema`
- `validation.isSuccess`
- `validation.totalEventsParsed`
- `validation.sampleEvents`
- `validation.errorMessage`
- `validation.validationErrors`

## Local Testing: Copy-Paste Commands

⚠️ **PowerShell and `&` in URLs:** PowerShell treats bare `&` as a background-job operator. Always build the body with `@{} | ConvertTo-Json` (as shown below) rather than an inline JSON string. This avoids escaping issues and works reliably with query-string URLs.

### Test-Fetch (PowerShell)

```powershell
# PowerShell — ICS test-fetch
# Use ConvertTo-Json to avoid & escaping issues with query-string URLs
$body = @{
    name        = "My ICS Source"
    description = "Community events ICS feed"
    type        = "Ics"
    feedUrl     = "https://example.org/calendar?catID=1&feed=calendar"
    schemaDefinition = "{}"
    metadata    = @{
        location = "Example City, WA"
        region   = "WA"
        category = "community"
        language = "en"
    }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "https://dev.api.wheneber.com/api/source-schemas/test-fetch" `
    -Method Post -Headers @{ Authorization = "Bearer <signed-in-access-token>" } `
    -Body $body -ContentType "application/json" -TimeoutSec 90
```

### Optional: Enable HTML Detail Enrichment In schemaDefinition

If your source keeps richer details on event pages, configure enrichment directly in `schemaDefinition`.

```json
{
  "eventEnrichment": {
    "enabled": true,
    "parser": "GenericHtml",
    "maxFetchesPerRun": 5,
    "sameHostOnly": true
  }
}
```

Notes:

- `maxFetchesPerRun` should stay small for test-fetch (for example `3` to `5`).
- `sameHostOnly: true` is recommended for safety.
- When `parser` is `HtmlLite`, `htmlLiteMappings` is required.

### Community Submission (PowerShell)

Run this after test-fetch passes with a good event count.

```powershell
# PowerShell — community-submissions (after test-fetch passes)
$schemaDef = @{
    validation = @{
        requiredFields     = @("title", "startTime")
        minEventsPerFetch  = 1
        maxEventsPerFetch  = 10000
    }
} | ConvertTo-Json -Depth 5 -Compress

$body = @{
    name        = "My ICS Source"
    description = "Community events ICS feed"
    type        = "Ics"
    feedUrl     = "https://example.org/calendar?catID=1&feed=calendar"
    schemaDefinition = $schemaDef
    metadata    = @{
        location          = "Example City, WA"
        region            = "WA"
        category          = "community"
        language          = "en"
        estimatedEventCount = 50
    }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "https://dev.api.wheneber.com/api/source-schemas/community-submissions" `
    -Method Post -Headers @{
        Authorization = "Bearer <signed-in-access-token>"
        "Idempotency-Key" = "example-submission-0001"
    } -Body $body -ContentType "application/json" -TimeoutSec 90
```

## Shared Validation Rules

The backend enforces these common rules for every source type:

- `schemaDefinition` must be valid JSON
- `schemaDefinition` cannot exceed the max depth limit of 30
- `mappings` cannot exceed 300 entries when present
- `requestWorkflow.steps` cannot exceed 25 entries when present
- `requestHeaders` cannot exceed 50 entries when present
- outbound URLs must be public `https`
- localhost, private-network, link-local, cloud-metadata, and restricted hosts are blocked after DNS resolution as well as before connecting
- redirects are disabled and pagination/detail/auth-token requests are bounded; contributor schemas must not configure authentication or request headers
- event/detail fetch links must remain on the submitted feed origin

Field precedence expectation for enrichment:

- detail value first
- calendar default second
- empty/null last

## Geolocation Mapping And Fallback Expectations

Use this guidance when authoring schemas for any source type.

- If the source provides valid coordinates, map and preserve that source geolocation evidence.
- If source coordinates are missing or invalid, ingestion falls back to geocoding from text fields such as `location` and `venueAddress`.
- Address query normalization is applied before geocoding, so cleaner `location` and `venueAddress` mappings improve results.
- On updates, if an incoming event has no geometry, existing persisted geometry is retained rather than cleared.

Coordinate validation guidance for contributors and AI:

- Latitude must parse as a number in range `-90` to `90`.
- Longitude must parse as a number in range `-180` to `180`.
- Coordinates with both values missing, non-numeric, or out of range should be treated as invalid and should trigger fallback geocoding.
- If source payload uses GeoJSON-style coordinate arrays, remember array order is usually longitude, latitude.

Common source key patterns to look for during mapping:

- latitude-like keys: `lat`, `latitude`, `geo.lat`, `location.latitude`, `geometry.location.lat`
- longitude-like keys: `lng`, `lon`, `long`, `longitude`, `geo.lng`, `location.longitude`, `geometry.location.lng`
- coordinate objects or arrays: `coordinates`, `geo`, `geometry`, `point`

Authoring note:

- Do not invent coordinates.
- If explicit coordinate mapping is not supported for a source shape, provide high-quality `location` and `venueAddress` fields for geocoding fallback.

## JsonApi Additional Contract Rules

`JsonApi` has extra schema-contract validation:

- `schemaVersion` must be `3`
- `pipeline.calendar` and `pipeline.event` are required
- calendar stage parser options belong under `pipeline.calendar.parser`
- stage fetch options belong under `pipeline.calendar.fetch` and `pipeline.event.fetch`
- `requestWorkflow` and `pagination` cannot both be configured in the same stage
- `pagination.maxRequestsPerRun` must be greater than `0` when provided

## Type-Specific Minimum Shapes

### Ics

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Ics",
      "parser": {
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

### Rss

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Rss",
      "parser": {
        "mappings": {
          "title": "title",
          "startTime": "pubDate",
          "eventUrl": "link",
          "id": "guid"
        },
        "filters": {
          "includeAny": [
            { "field": "title", "value": "event", "matchType": "contains" }
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
    },
    "event": {
      "type": "None",
      "input": { "mode": "none" },
      "parser": {}
    }
  }
}
```

### JsonApi

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

When the source splits date/time fields or exposes source timezone, place transforms in `pipeline.calendar.parser.responseTransforms`.
For historical v1/v2 examples, see the legacy JsonApi v2 reference page.

### HtmlLite

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "HtmlLite",
      "parser": {
        "eventCardSelector": "div.event-card",
        "mappings": {
          "title": "h3",
          "startTime": "time::attr(datetime)",
          "url": "a::attr(href)"
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

## RSS Default Validation And Filtering Rules

Use these defaults for new RSS source schemas unless the source proves a tighter rule is safe:

```json
{
  "validation": {
    "minEventsPerFetch": 1,
    "requiredFields": ["title", "startTime"]
  }
}
```

Authoring expectations:

- Always require `title` and `startTime` for RSS drafts.
- Add `id` mapping when the feed exposes a stable source identifier (typically mapped from `guid`).
- Add `eventUrl` mapping when the feed exposes event detail pages (typically mapped from `link` or `url`).
- Add `location` or `endTime` to `requiredFields` only when the feed is consistently populated and you intentionally want items missing those fields dropped.
- Do not rely on `maxEventsPerFetch` as a content filter; only set it when you have a verified operational reason to bound expected feed size.

Filtering expectations for RSS:

- Prefer source-side filtering first: pick the right upstream feed URL and use any upstream category/date/event-only query options the publisher already exposes.
- When source-side filtering is insufficient, use `pipeline.calendar.parser.filters.includeAny` / `excludeAny`.
- RSS filter rules must be deterministic and use supported fields only: `title`, `description`, `eventUrl`, `category`.
- If the feed mixes blog/news/non-event items and you cannot narrow it at the source, do not treat it as a valid RSS onboarding target.
- If detail enrichment is required, `eventUrl` becomes operationally required even though it is not part of `requiredFields`.

## Common Invalid Patterns

These mistakes are common when generating schemas with chat models.

### Invalid: unsupported HtmlLite selector lists

```json
{
  "eventCardSelector": ".event-card, .featured-event",
  "mappings": {
    "title": ".event-title, h3, h2"
  }
}
```

Why invalid:
- HtmlLite does not support comma-separated selector lists.

Valid rewrite:

```json
{
  "eventCardSelector": "xpath=.//*[contains(@class,'event-card') or contains(@class,'featured-event')]",
  "mappings": {
    "title": "xpath=.//*[contains(@class,'event-title') or self::h3 or self::h2][1]"
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

### Invalid: unsupported HtmlLite pagination shape

```json
{
  "pagination": {
    "enabled": true,
    "nextPageSelector": "a.next"
  }
}
```

Valid rewrite:

```json
{
  "pagination": {
    "type": "nextLink",
    "nextPageSelector": "a.next@href",
    "maxPages": 5
  }
}
```

### Invalid: unrecognized HtmlLite fields

```json
{
  "baseUrl": "https://example.org",
  "transforms": [
    {
      "field": "url",
      "type": "absoluteUrl"
    }
  ]
}
```

Why invalid:
- `baseUrl` and `transforms` are not part of the HtmlLite contract.
- `url` and `imageUrl` are already resolved to absolute URLs downstream when valid relative links are captured.

Use `fieldTransforms` for low-risk per-field cleanup instead of top-level `transforms`.

Supported low-risk HtmlLite transform types:

- `trim`
- `collapseWhitespace`
- `removePrefix`
- `removeSuffix`
- `replaceLiteral`
- `stripWrappingQuotes`
- `nullIfEqualsAny`
- `regexReplace`

Use `regexReplace` only for deterministic cleanup (for example, removing a trailing second date from a multi-date field). Avoid cross-field reconstruction or expression-eval style transforms.

See [HtmlLite Schema Basics](html-lite/schema-basics.md#field-transforms-low-risk) and [HtmlLite Time Handling](html-lite/time-handling.md#multi-date-strings--extracting-the-first-date-only) for examples.

### Invalid: unsupported `regex:` mapping syntax

```json
{
  "mappings": {
    "startTime": "regex:(\\w+ \\d{1,2}, \\d{4})"
  }
}
```

Why invalid:
- HtmlLite mappings expect supported selector strings, XPath selectors, attribute extraction syntax, or `literal:` values.
- `regex:` is not part of the HtmlLite mapping DSL.

Use selector-based extraction instead (for example `time::attr(datetime)` or split `startDate` and `startTime`).

### Invalid: wrong field name

```json
{
  "detailPage": {
    "enabled": true,
    "detailMappings": {
      "address": ".event-address"
    }
  }
}
```

Valid rewrite:

```json
{
  "detailPage": {
    "enabled": true,
    "detailMappings": {
      "venueAddress": ".event-address"
    }
  }
}
```

## Validation Pass Criteria

Treat a draft as ready for admin review only when:

1. `validation.isSuccess` is `true`
2. `validation.totalEventsParsed` is non-zero and plausible
3. `validation.sampleEvents` show good coverage for title, time, URL, and location when available
4. the source type is still the best available source after testing

## Iteration Strategy For Any Source Type

Use a strict one-change-at-a-time loop:

1. Start with the smallest valid schema shape for the chosen type.
2. Run `test-fetch` and capture `isSuccess`, `totalEventsParsed`, `sampleEvents`, and `errorMessage`.
3. Change only one dimension per retry (selector/path, mapping, pagination, headers, or auth).
4. Re-run `test-fetch` immediately after each change.
5. Stop iterating when results are stable across repeated runs.

This avoids masking root causes and makes failures easier to explain in handoff.

## JsonApi: Zero-Event Diagnostic Order

When `validation.isSuccess = true` but `totalEventsParsed = 0` (or implausibly low), check in this order:

1. `feedUrl` returns the expected JSON payload at request time.
2. `eventArrayPath` points to repeated event items, not wrapper/config nodes.
3. If `responseTransforms` are used, `responseTransforms.flattenPath` points to the intended repeated event nodes.
4. Mapping paths are relative to each event item.
5. Required fields remain populated after mappings/transforms; zero parsed events can mean every candidate event was filtered out because `title` or `startTime` ended up empty.
6. If the source provides timezone, `timeZone` is mapped from source rather than assumed from runtime environment.
7. Required headers are present (for example, browser-like `User-Agent` when needed).
8. Pagination/workflow settings are not limiting extraction unexpectedly.

If a payload clearly contains events but extraction is empty, simplify `eventArrayPath` and mappings until one sample event parses, then build back up.

## Formal Schema Definition Status

Wheneber now publishes contributor-facing JSON Schema documents for all supported `schemaDefinition` types.

See [Source Schema JSON Schema Files](json-schema-files.md).

Use these as the source of truth instead:

- the contributor-facing JsonApi v2 JSON Schema file for machine-readable authoring help
- the contributor-facing source-schema JSON Schema files for machine-readable authoring help
- the type-specific documentation on this site
- backend contract validation and error messages
- the backend source models and tests for supported fields and combinations

That means contributors and AI should treat the JSON Schema plus docs as authoring guidance, while backend validation remains the authoritative final contract.

## Contributor Handoff Checklist

Include:

- `sourceSchema.id`
- `feedUrl`
- final `schemaDefinition`
- validation summary
- any known limitations such as missing description, incomplete location, or pagination constraints

For `JsonApi`, also include:

- confirmed event collection path used in production schema
- any required headers or token dependencies
- parser or mapping limitations discovered during iteration
- geolocation note: `source-coordinates-used` or `geocode-fallback-expected`