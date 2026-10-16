# v3 Event Input Constraints

The backend HTML/JSON detail executor supports:

- `none`: no detail stage.
- `calendarFieldUrl`: read the URL from the named calendar field.
- `fixedUrl`: substitute calendar fields into `urlTemplate`.

For example, a listing link `/events/123` can be made absolute without a backend
URL-resolution change:

```json
"input": {
  "mode": "fixedUrl",
  "urlTemplate": "https://example.org{{eventUrl}}"
}
```

`calendarFieldUrl` requires an already absolute HTTP(S) URL in its field:

```json
"input": { "mode": "calendarFieldUrl", "field": "eventUrl" }
```

This backend executor does not resolve relative links using `baseUrl` or the feed
URL. Do not assume older contributor test-fetch/scaffolding behavior applies to
stored-schema tests and worker execution.

Both modes apply `limits.sameHostOnly` (default `true`) against the feed host
before detail requests. Non-HTTP(S) URLs are rejected. `maxRequestsPerRun` bounds
HTML fetch attempts, including failures. These checks do not implement a new
redirect-host policy; redirects retain the existing HTTP client behavior.

The same effective calendar headers, restricted-header exclusions, timeout, and
response-size bounds are used for listing and detail requests in stored-schema
tests and runtime workers. See [HTML detail extraction](html-detail-enrichment.md)
for occurrence scoping, transforms, fallback, identity, and minimum-count semantics.
