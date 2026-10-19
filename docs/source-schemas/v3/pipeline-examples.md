# Source Schema v3 Pipeline Examples

## Minimal Valid Schemas by Type

Use these templates as a starting point. Add fields incrementally and validate after each change.

### Html -> Html (HtmlLite Sources — Minimum Valid v3 Pipeline)

Use this pipeline pair for **HtmlLite** sources. For v3, `Html -> None` is not allowed by the current security allowlist.

!!! warning "pipeline.calendar.type for HtmlLite sources"
    The **top-level source `type`** field in the submission envelope is `"HtmlLite"`.

    The **inner `pipeline.calendar.type`** inside `schemaDefinition` must be `"Html"` — **not** `"HtmlLite"`. Using `"HtmlLite"` as the inner calendar type fails v3 runtime validation.

> Note: During current phase-2 scaffolding, `pipeline.event.input.field` must be `eventUrl`.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "fetch": {
        "requestHeaders": {
          "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
        }
      },
      "parser": {
        "eventCardSelector": ".event-item",
        "mappings": {
          "title": "h2",
          "startTime": ".date",
          "eventUrl": "a@href"
        }
      }
    },
    "event": {
      "type": "Html",
      "input": { "mode": "calendarFieldUrl", "field": "eventUrl" },
      "parser": { "mappings": {} },
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

### Ics -> None

Use `Ics -> None` when the ICS feed provides complete event data and no HTML detail-page enrichment is needed. This is the correct combination for most ICS sources.

> **Common mistake:** Setting `event.type` to `"Html"` for a plain ICS feed that requires no detail-page fetch will fail validation. Use `"None"` unless you need to follow `eventUrl` to fetch per-event HTML pages.

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
      "input": {
        "mode": "none"
      },
      "parser": {}
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

### Ics -> None (with fieldTransforms)

Use this variant when ICS fields are mostly correct but require deterministic cleanup (for example, Teams links prefixed in `LOCATION`).

```json
{
  "schemaVersion": 3,
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
    },
    "event": {
      "type": "None",
      "input": {
        "mode": "none"
      },
      "parser": {}
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

> `fieldTransforms` is optional and should be used for cleanup only, not for reconstructing missing required fields.

## Ics -> Html

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
          "description": ".event-details",
          "venueAddress": ".venue-address",
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

## Json -> Json

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Json",
      "parser": {
        "eventArrayPath": "$.events[*]",
        "mappings": {
          "title": "$.name",
          "startTime": "$.start",
          "eventUrl": "$.detailUrl"
        }
      }
    },
    "event": {
      "type": "Json",
      "input": {
        "mode": "calendarFieldUrl",
        "field": "eventUrl"
      },
      "parser": {
        "eventArrayPath": "$.items[*]",
        "mappings": {
          "description": "$.description",
          "venueName": "$.venue.name",
          "venueAddress": "$.venue.address"
        }
      },
      "limits": {
        "sameHostOnly": true,
        "maxRequestsPerRun": 10
      }
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

## Json -> Html (Cross-Host Detail Description)

Use this pattern when:

- list API host differs from public detail-page host
- list payload has sparse or empty `description`
- detail HTML contains full event body

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Json",
      "fetch": {
        "method": "Post"
      },
      "parser": {
        "eventArrayPath": "$.hits[*]",
        "mappings": {
          "id": "$.objectID",
          "title": "$.title",
          "startTime": "$.dates[*]",
          "eventUrl": "$.url",
          "venueName": "literal:The Museum",
          "venueAddress": "literal:1945 SE Water Ave, Portland, OR 97214",
          "location": "literal:1945 SE Water Ave, Portland, OR 97214"
        }
      }
    },
    "event": {
      "type": "Html",
      "input": {
        "mode": "calendarFieldUrl",
        "field": "eventUrl",
        "baseUrl": "https://omsi.edu"
      },
      "parser": {
        "mappings": {
          "description": ".container-content"
        }
      },
      "limits": {
        "sameHostOnly": false,
        "maxRequestsPerRun": 25
      }
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime"]
  }
}
```

Notes:

- `sameHostOnly: false` is required when list and detail hosts differ.
- `input.baseUrl` ensures detail links resolve to the public content host.
- detail values override defaults only when detail values are non-empty.

## Html -> Html with Multi-Date Expansion

Use this when one HTML event card contains multiple occurrence dates in one text field.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Html",
      "parser": {
        "eventCardSelector": ".event-card",
        "mappings": {
          "id": "a@href",
          "title": "h3",
          "startTime": ".showings",
          "url": "a@href",
          "timeZone": "literal:America/Los_Angeles"
        },
        "multiDateExpansion": {
          "enabled": true,
          "sourceField": "startTime",
          "defaultTime": "7:30 PM",
          "defaultTimeZone": "America/Los_Angeles",
          "fallbackMode": "legacySingle",
          "limits": {
            "maxOccurrencesPerCard": 12,
            "maxExpandedEventsPerRun": 500,
            "maxParseFailuresPerCard": 3
          }
        },
        "detailPage": {
          "enabled": true,
          "linkSelector": "a@href",
          "occurrenceSelector": ".occurrence",
          "occurrenceMappings": {
            "startTime": ".occ-start::attr(datetime)",
            "endTime": ".occ-end::attr(datetime)"
          }
        }
      }
    },
    "event": {
      "type": "Html",
      "input": { "mode": "calendarFieldUrl", "field": "eventUrl" },
      "parser": { "mappings": {} },
      "limits": {
        "sameHostOnly": true,
        "maxRequestsPerRun": 10
      }
    }
  },
  "validation": {
    "requiredFields": ["title", "startTime", "url"]
  }
}
```

Composition rule:

1. `multiDateExpansion` determines how many events are emitted from each card.
2. `detailPage.occurrenceSelector` refines each emitted event.
3. Detail occurrence matching does not change event count.
## Generic JSON POST calendar

Anonymous read-only JSON queries can use a typed body. This complete example
uses query pagination; the literal body remains identical on every page.
See [JSON API request policy](../json-api.md#anonymous-json-post-calendars) for
limits and contributor restrictions. This does not discover or batch dates.

```json
{
  "schemaVersion": 3,
  "pipeline": {
    "calendar": {
      "type": "Json",
      "fetch": {
        "method": "POST",
        "jsonBody": { "categoryID": 0, "includePast": false, "filters": [] },
        "pagination": {
          "strategy": "Page",
          "pageParamName": "page",
          "totalPagesPath": "$.pages"
        }
      },
      "parser": {
        "eventArrayPath": "$.events[*]",
        "mappings": {
          "id": "$.id",
          "title": "$.title",
          "startTime": "$.start",
          "location": "$.venue",
          "description": "$.description",
          "imageUrl": "$.image",
          "eventUrl": "$.url"
        }
      }
    },
    "event": { "type": "None", "input": { "mode": "none" }, "parser": {} }
  }
}
```
