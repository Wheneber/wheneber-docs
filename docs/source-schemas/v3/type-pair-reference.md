# v3 Calendar/Event Type Pair Reference

v3 accepts only specific `pipeline.calendar.type -> pipeline.event.type` combinations.

## Allowed pairs

- `Ics -> None`
- `Ics -> Html`
- `Ics -> Json`
- `Html -> Html`
- `Html -> Json`
- `Html -> None` **only** with an allowlisted code-owned calendar adapter
  (`eventon`, `ticketweb`, `ticketmaster`, `tockify`, or `journey`)
- `Json -> None`
- `Json -> Html`
- `Json -> Json`
- `Rss -> None`
- `Rss -> Html`
- `Rss -> Json`

Any pair outside this list fails schema admission.

## Important implications

- `Html -> None` is rejected for normal parser execution. Code-owned complete
  projection adapters require this pair; their calendar/event parsers must be
  empty and no separate detail stage is permitted.
- `Ics -> None` and `Rss -> None` remain valid when calendar stage already contains needed fields.
- `Json -> None` is valid when calendar payload already includes full event fields.

## Recommendation

Choose the simplest allowed pair that still gives complete and accurate event data.

- If detail fetch is unnecessary and your type supports `-> None`, use it.
- If detail fetch is needed, use `-> Html` or `-> Json` with conservative fetch limits.
