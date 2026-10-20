# Ics Source Guide

Use `Ics` when the source publishes a valid iCalendar feed. This is usually the best option when available because the parser reads standard calendar fields directly.

For a machine-readable authoring contract, use the contributor-facing [Ics JSON Schema](ics.schema.json).

All ingestion follows the [shared schedule policy](schedule-policy.md). A timed
DTSTART without DTEND/DURATION receives one elapsed hour with a final description
disclaimer. DATE DTSTART without an end is all-day to the next source-local
midnight (DST-aware). Published DTEND remains exclusive and DURATION wins over
the default. Missing/invalid DTSTART is a failure, never publication/current time.

## ⚠️ ONLY Valid Format For ICS

The `Ics` source schema submission to Wheneber **has exactly this structure and no other**:

```json
{
  "name": "<source name>",
  "description": "<source description>",
  "type": "Ics",
  "feedUrl": "<url-that-returns-ics-feed>",
  "schemaDefinition": "<json-string-with-only-validation>",
  "metadata": {
    "location": "<location>",
    "category": "<category>",
    "region": "<region>",
    "eventStartNotBeforeDate": "<yyyy-mm-dd>",
    "eventFallbackUrl": "<public-events-page-url>",
    "customAttributes": {
      "defaultLocation": "<street-city-state-zip>",
      "defaultVenueAddress": "<street-city-state-zip>"
    }
  }
}
```

The `schemaDefinition` is a compact JSON string containing optional `validation`, optional `eventEnrichment`, and optional `fieldTransforms`:

```json
{
  "validation": {
    "requiredFields": ["title", "startTime"],
    "minEventsPerFetch": 1,
    "maxEventsPerFetch": 10000
  },
  "eventEnrichment": {
    "enabled": true,
    "parser": "GenericHtml",
    "maxFetchesPerRun": 5,
    "sameHostOnly": true
  },
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
```

or for test-fetch, simply:

```json
{}
```

**Forbidden in Ics request/schema contracts:** top-level `url`, `mappings`, `eventMapping`, top-level `transforms`, `recurrence`, `sourceType`, `eventDefaults`, `allDay`.

`fieldTransforms` is supported inside `schemaDefinition` for deterministic per-field cleanup after RFC 5545 parsing.

## When To Use It

Use `Ics` when:

- the URL returns a `.ics` file or `text/calendar`
- the feed already includes title, start, end, location, description, or event URL
- the source has stable event UIDs

Do not use `Ics` when you only have HTML or JSON.

## ICS Field Mapping

The parser reads standard RFC 5545 VEVENT fields automatically. No `mappings` configuration is needed or supported.

| ICS VEVENT Field | Maps To | Notes |
|---|---|---|
| `SUMMARY` | `title` | Required. Events without SUMMARY are skipped. |
| `DTSTART` | `startTime` | Required. Missing/invalid DTSTART fails rather than inventing a date. |
| `DTEND` | `endTime` | Optional, authoritative exclusive end. Without DTEND/DURATION: timed start + 1 elapsed hour with disclosure, or next local midnight for DATE. |
| `DURATION` | `endTime` | Publisher duration when DTEND is absent; date-only days/weeks use local calendar boundaries. |
| `DESCRIPTION` | `description` | Optional. If `URL` is absent or resolves back to the source feed, the first http/https URL found in DESCRIPTION is used as `eventUrl`. |
| `LOCATION` | `location` | Optional. Used for geocoding fallback when coordinates are absent. |
| `URL` | `eventUrl` | Optional. Relative URLs are resolved against the source feed. If the resolved URL points back to the source feed, it is ignored and DESCRIPTION fallback is used instead. |
| `UID` | identity key | Strongly recommended. Improves deduplication stability. |
| `TZID` (on DTSTART) | `timeZone` | Carried through from the timezone identifier on the start field. |

> **Note on `URL` vs `DESCRIPTION`:** Some sources use the `URL` VEVENT field for a shared category link rather than a per-event deep link. The parser resolves relative URLs against the source feed and ignores them when they normalize to the same target as the feed itself, even if query parameter order differs. In that case, if the `DESCRIPTION` field contains a bare absolute URL, the parser uses it as `eventUrl` automatically. You do not need to configure this — it is built-in fallback behavior.

## Optional Field Transforms (Ics)

Use `fieldTransforms` for deterministic cleanup of already-parsed ICS fields. This does not replace RFC 5545 parsing and does not introduce mappings.

Supported keys are parsed event fields such as:

- `title`
- `description`
- `location`
- `venueName`
- `venueAddress`
- `url`
- `imageUrl`

Example: strip Teams URL from `location` while preserving venue text.

```json
{
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
```

## Test-Fetch Example

```json
{
  "type": "Ics",
  "feedUrl": "https://example.org/events.ics",
  "schemaDefinition": "{}",
  "metadata": {
    "eventStartNotBeforeDate": "2026-01-01",
    "eventFallbackUrl": "https://example.org/events"
  }
}
```

## Optional: HTML Detail Enrichment For ICS

Some ICS feeds provide only partial event detail in VEVENT fields, while full details live on an event page linked by `eventUrl`.

Wheneber supports optional HTML detail enrichment for `Ics` sources during ingestion and test-fetch.

- The feed remains the calendar source of truth.
- Detail pages are fetched only when enrichment is enabled.
- Enrichment fills weak or missing fields such as `description`, `venueAddress`, `imageUrl`, and (when missing) `location`.
- Strong ICS fields (`title`, `startTime`, `endTime`, `UID`) are not overridden.

Enable with schema-level configuration in `schemaDefinition`:

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

Use `parser: "HtmlLite"` only when you also provide `htmlLiteMappings`.

Recommended starting values:


- `enabled = true`
- `maxFetchesPerRun = 25` for ingestion
- `sameHostOnly = true`

For test-fetch parity with runtime ingestion behavior, send the same `eventEnrichment` block in the payload used for `POST /api/source-schemas/test-fetch`.

## Wrong Vs Right Payload Shape

The `Ics` `schemaDefinition` supports `validation`, optional `eventEnrichment`, and optional `fieldTransforms`. The parser still reads RFC 5545 fields (`UID`, `SUMMARY`, `DTSTART`, `DTEND`, `DESCRIPTION`, `LOCATION`, `URL`) directly from the feed. Do not add `mappings` or `eventMapping`.

Wrong — top-level `url` instead of `feedUrl`, `eventMapping` inside `schemaDefinition`:

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

Also wrong — `mappings` is equally unsupported in `schemaDefinition`:

```json
{
  "type": "ics",
  "url": "https://calendar.google.com/calendar/ical/lacentercalendar%40gmail.com/public/basic.ics",
  "mappings": {
    "id": "uid",
    "title": "summary",
    "description": "description",
    "location": "location",
    "startTime": "dtstart",
    "endTime": "dtend",
    "allDay": "allDay"
  }
}
```

Right for this API contract (minimal `schemaDefinition`):

```json
{
  "type": "Ics",
  "feedUrl": "https://calendar.google.com/calendar/ical/lacentercalendar%40gmail.com/public/basic.ics",
  "schemaDefinition": "{}"
}
```

Also valid when you want explicit validation guards and deterministic field cleanup:

```json
{
  "type": "Ics",
  "feedUrl": "https://calendar.google.com/calendar/ical/lacentercalendar%40gmail.com/public/basic.ics",
  "schemaDefinition": "{\"validation\":{\"requiredFields\":[\"title\",\"startTime\"],\"minEventsPerFetch\":1},\"fieldTransforms\":{\"location\":[{\"type\":\"regexReplace\",\"value\":\"^\\\\s*https://teams\\\\.microsoft\\\\.com[^,]*(?:,\\\\s*)?(.*)$\",\"replacement\":\"$1\"},{\"type\":\"trim\"},{\"type\":\"collapseWhitespace\"}]}}"
}
```

Summary of rules:

| Field | Correct location | Notes |
|---|---|---|
| `feedUrl` | top-level request field | not `url` |
| `type` | top-level request field | value is `Ics` (capital I) |
| `schemaDefinition` | top-level request field | compact JSON string |
| `validation` | inside `schemaDefinition` | optional validation constraints |
| `eventEnrichment` | inside `schemaDefinition` | optional detail-page enrichment settings |
| `fieldTransforms` | inside `schemaDefinition` | optional per-field cleanup after parse |
| `mappings` | **nowhere** | unsupported for `Ics` |
| `eventMapping` | **nowhere** | unsupported for `Ics` |
| `url` | **nowhere** | use `feedUrl` at top level |

## Community Submission Example

```json
{
  "name": "Example Community Center Events",
  "description": "Public iCalendar feed for Example Events",
  "type": "Ics",
  "feedUrl": "https://example.org/events.ics",
  "schemaDefinition": "{\"validation\":{\"requiredFields\":[\"title\",\"startTime\"]}}",
  "metadata": {
    "location": "Example City",
    "category": "community",
    "region": "WA",
    "eventStartNotBeforeDate": "2026-01-01",
    "eventFallbackUrl": "https://example.org/events",
    "customAttributes": {
      "defaultLocation": "123 Main St, Example City, WA 99999"
    }
  }
}
```

## Metadata Policy Notes For ICS

- `eventStartNotBeforeDate` filters out older events in both test-fetch and ingestion.
- `eventFallbackUrl` is used only when an event is missing a usable URL.
- `customAttributes.defaultLocation` fills only `location`.
- `customAttributes.defaultVenueAddress` fills only `venueAddress`.
- `defaultLocation` and `defaultVenueAddress` are independent and do not cross-fill.

## Requirements And Validation Notes

- `feedUrl` must return a valid ICS feed
- `title` and `startTime` should be present on real events
- `UID` is strongly preferred because it improves identity stability
- missing `DtEnd` values may be normalized to a default duration by the parser
- all-day and timezone data should be checked in `sampleEvents`

## Recommended Contributor Workflow

1. Confirm the feed is really ICS.
2. Start with the minimal validation object above.
3. Run `test-fetch`.
4. Check that titles and start times are populated and plausible.
5. Submit with `community-submissions`.

## Common Failure Modes

- The URL is not actually an ICS response
- The feed is empty or stale
- Titles are blank or missing on some events
- Timezone handling in the source is inconsistent

See also: [Submission API And Validation](submission-api-and-validation.md)

## Finding an ICS Feed from a Google Calendar Embed

Some sources do not publish their own ICS feed. Instead, their events page embeds a Google Calendar iframe. You can still create an `Ics` source by extracting the underlying feed URL from the embed.

Steps:

1. View the page source (`Ctrl+U` in most browsers — or use a "View page source" option).
2. Search for `calendar.google.com` and locate the `<iframe>` `src` attribute value.
3. The `src` URL contains one or more `src=...` query parameters. Each value is a base64-encoded Google Calendar ID.
4. Decode each base64 value. In PowerShell: `[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("PASTE_VALUE_HERE"))`. In a browser console: `atob("PASTE_VALUE_HERE")`. The decoded ID ends with `@group.calendar.google.com` or `@gmail.com`.
5. If the iframe contains multiple `src` parameters, inspect each decoded calendar ID before choosing one. Embedded pages often include unrelated calendars such as U.S. holidays, shared venue calendars, or secondary collections.
6. Submit only the venue-owned or organization-owned public calendar that actually represents the source you are onboarding.
7. URL-encode the decoded calendar ID (replace `@` with `%40`, `/` with `%2F`, etc.).
8. Construct the ICS feed URL: `https://calendar.google.com/calendar/ical/{URL_ENCODED_CALENDAR_ID}/public/basic.ics`
9. Test the URL in your browser or with `test-fetch`. A private or restricted calendar returns 404 or 403 — only submit public feeds.

When a source embeds multiple Google Calendars (for example, one for performances and one for auditions), create a **separate ICS source submission** for each public calendar.

## Using ICS Sources with the v3 Pipeline

When writing a v3 `schemaDefinition` for an ICS feed, use `pipeline.calendar.type = "Ics"`. If the feed provides complete event data without any detail-page enrichment, `pipeline.event.type` must be `"None"`:

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Ics",
      "parser": {}
    },
    "event": {
      "type": "None",
      "input": { "mode": "none" },
      "parser": {}
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

Do not set `pipeline.event.type` to `"Html"` for a plain ICS feed that needs no HTML enrichment — that combination requires a real HTML parser input configuration and will fail validation. Use `Ics → Html` only when you also need to fetch each event's detail page via `eventUrl`. See [v3 Pipeline Examples](v3/pipeline-examples.md) for the full matrix.