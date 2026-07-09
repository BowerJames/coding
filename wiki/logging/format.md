---
type: practice
title: Log line format
description: "All log output uses the `<timestamp_utc> <level> <message>` format."
tags: [logging, observability]
timestamp: 2026-07-09T13:00:43Z
---

Every log line emitted by code in this wiki's scope MUST use the format:

```
<timestamp_utc> <level> <message>
```

Three fields, space-delimited, in a fixed order.

# Schema

| Field           | Meaning                                                                                                                          |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `timestamp_utc` | When the event occurred, in UTC. Use ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`).                                                           |
| `level`         | One of `DEBUG`, `INFO`, `WARNING`, `ERROR` — the canonical set (see [Log output streams](streams.md)). Do not invent new levels. |
| `message`       | The human-readable description of the event.                                                                                     |

# Rationale

- **UTC, not local time.** Local time is ambiguous across servers and timezones; UTC makes logs from multiple sources comparable and trivially sortable. The field is named `timestamp_utc` to make the choice explicit rather than accidental.
- **Level is a first-class field.** Putting severity in a fixed position lets consumers filter (e.g. `grep ERROR`) without parsing freeform text.
- **Fixed field order.** A predictable structure means logs are greppable, diffable, and machine-parseable without bespoke tooling.

# Examples

```
2026-07-09T10:56:29Z INFO User 42 authenticated successfully
2026-07-09T10:56:30Z WARNING Retrying request after timeout (attempt 2/5)
2026-07-09T10:57:01Z ERROR Database connection refused
```

# Notes

This format is deliberately minimal — a single freeform `<message>` tail. That keeps the rule simple and universal across languages, but it leaves **no dedicated position for structured context** (request IDs, user IDs, span IDs, error codes). Such context can still be embedded inside `<message>`, but it will not occupy a fixed, queryable field.

When richer observability is needed (e.g. distributed tracing, structured/JSON logging with first-class fields), that belongs in a *separate*, more detailed practice — this page covers the baseline format only. Further logging pages already live under `/logging/` (see [Log output streams](streams.md) and [Logging in Python](/python/logging.md)); link any new ones in `# See also` below.

# See also

- [Log output streams](streams.md) — which stream (`stdout`/`stderr`) each level routes to, and the alerting rationale.
- [Logging in Python](/python/logging.md) — language-specific implementation of this format plus the stream routing.

# Citations

[1] Ingested convention directive: "All logging should be done in `<timestamp_utc> <level> <message>` format." Personal engineering convention; no external URL. Ingested 2026-07-09.
