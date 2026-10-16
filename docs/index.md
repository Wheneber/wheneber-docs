# Wheneber Source Onboarding Docs

This site is the public source-of-truth for onboarding event sources into Wheneber.

It is intentionally split into two execution tracks:
- Human source contributors
- Autonomous AI agents

> Scope boundary: these docs cover source onboarding and source schema submission only. They do not cover code contribution to private Wheneber application repositories.

## Machine-Readable Access

AI tools and scrapers: the rendered site requires no JavaScript (MkDocs generates static HTML), but the most reliable way to load these docs programmatically is via raw GitHub markdown.

- **Full page index:** [`/llms.txt`](https://Wheneber.github.io/wheneber-docs/llms.txt) — lists every page and schema file with a direct raw GitHub link
- **Raw markdown base URL:** `https://raw.githubusercontent.com/Wheneber/wheneber-docs/main/docs/`
- **JSON Schema files** (machine-readable `schemaDefinition` contracts): see [Source Schema JSON Schema Files](source-schemas/json-schema-files.md)

Example — fetch the ICS guide as plain text:
```
https://raw.githubusercontent.com/Wheneber/wheneber-docs/main/docs/source-schemas/ics.md
```

## Choose A Track

### Human Source Contributors

Start here:
1. [Choose A Source Type](source-schemas/choose-source-type.md)
2. [Submission API And Validation](source-schemas/submission-api-and-validation.md)
3. [Ics Source Guide](source-schemas/ics.md)
4. [Rss Source Guide](source-schemas/rss.md)
5. [JsonApi Source Guide](source-schemas/json-api.md)
6. [HtmlLite Overview](source-schemas/html-lite/overview.md)

Use this track when a person is inspecting feeds, choosing the right source type, refining schemaDefinition, and submitting drafts.

### Autonomous AI Agents

Start here:
1. [Autonomous Runbook](ai/source-onboarding/runbook.md)
2. [Choose A Source Type](source-schemas/choose-source-type.md)
3. [Submission API And Validation](source-schemas/submission-api-and-validation.md)
4. [ChatGPT Prompt (No HTTP tool use)](ai/source-onboarding/prompt-template.md)

Use this track when an agent can browse pages and execute API calls end-to-end, or when a chat model must generate a human-runnable submission script.

## Source Onboarding Lifecycle

Contract reminder for ICS onboarding:
- use top-level `type`, `feedUrl`, and `schemaDefinition`
- do not use top-level `url` or `eventMapping` for this API
- keep `Ics` `schemaDefinition` minimal by default (`{}` or validation-only), and add `fieldTransforms` only when deterministic cleanup is needed

1. Discover a starting URL and identify the most stable source type.
2. Build the smallest valid `schemaDefinition` for that type.
3. Validate with `POST /api/source-schemas/test-fetch`.
4. Submit a contributor draft with `POST /api/source-schemas/community-submissions`.
5. Re-test and re-submit until validation is stable and non-zero.
6. Hand off schema ID and validation notes for admin review.

## Supported Source Types

| Type | Use It When | Avoid It When | Primary Reference |
|---|---|---|---|
| `Ics` | The source publishes a valid iCalendar or `.ics` feed | The source only offers HTML or JSON | [Ics Source Guide](source-schemas/ics.md) |
| `Rss` | The source publishes RSS or Atom where each item represents an event | The feed is not event-oriented or lacks usable dates | [Rss Source Guide](source-schemas/rss.md) |
| `JsonApi` | The site exposes event data through JSON endpoints | Static HTML or ICS already provides the same data more simply | [JsonApi Source Guide](source-schemas/json-api.md) |
| `HtmlLite` | The site renders event cards in static HTML without JavaScript-only hydration | The page is JS-rendered and there is a better structured feed | [HtmlLite Overview](source-schemas/html-lite/overview.md) |

## Core Reference Pages

| Area | Page | Purpose |
|---|---|---|
| Shared | [Choose A Source Type](source-schemas/choose-source-type.md) | Decision guide for picking the right parser and schema contract |
| Shared | [Submission API And Validation](source-schemas/submission-api-and-validation.md) | Common payload envelope, test-fetch flow, and contributor-safe endpoints |
| Shared | [Source Schema JSON Schema Files](source-schemas/json-schema-files.md) | Machine-readable schemaDefinition contracts for all supported source types |
| Adapter | [Tockify Public Calendar](source-schemas/v3/tockify-adapter.md) | Paginated public inventory, rich detail joins, epoch schedules and bounded execution |
| Shared | [Validation Checklist](source-schemas/html-lite/validation-checklist.md) | Field-completeness and handoff readiness gates |
| Shared | [Troubleshooting](source-schemas/html-lite/troubleshooting.md) | Recovery order for zero-parse and validation failures |
| Ics | [Guide](source-schemas/ics.md) | When to use ICS, minimal schemaDefinition, optional fieldTransforms, and validation notes |
| Rss | [Guide](source-schemas/rss.md) | RSS extraction rules and contributor workflow |
| JsonApi | [Guide](source-schemas/json-api.md) | JSON path mappings, pagination, auth, and advanced validation rules |
| HtmlLite | [Overview](source-schemas/html-lite/overview.md) | Contract summary and deterministic build order |
| HtmlLite | [API Workflow](source-schemas/html-lite/api-workflow.md) | Draft submission behavior and response interpretation |
| HtmlLite | [Schema Basics](source-schemas/html-lite/schema-basics.md) | Minimal schema and progressive enrichment pattern |
| HtmlLite | [Selectors](source-schemas/html-lite/selectors.md) | CSS-like and XPath usage rules |
| HtmlLite | [Event Identity](source-schemas/html-lite/event-identity.md) | Stable ID strategy and fallback behavior |
| HtmlLite | [Time Handling](source-schemas/html-lite/time-handling.md) | Date/time parsing and timezone assumptions |
| HtmlLite | [Platform Patterns](source-schemas/html-lite/platform-patterns.md) | Platform-specific onboarding recipes for common builders such as Squarespace and Wix |
| HtmlLite | [Pagination](source-schemas/html-lite/pagination.md) | Multi-page extraction strategies |
| HtmlLite | [Detail Pages](source-schemas/html-lite/detail-pages.md) | Detail-page enrichment and merge rules |
| HtmlLite | [Validation Checklist](source-schemas/html-lite/validation-checklist.md) | Quality gates before admin handoff |
| HtmlLite | [Troubleshooting](source-schemas/html-lite/troubleshooting.md) | Fast diagnosis for zero-parse and validation failures |
