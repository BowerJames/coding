---
type: practice
title: Log line format
description: "Every log line carries reserved core fields (UTC timestamp, level, message) and supports scoped structured context; the wire format is tailorable to the application (plain text or JSON)."
tags: [logging, observability]
timestamp: 2026-07-23T00:00:00Z
---

Every log line emitted by code in this wiki's scope MUST carry a small set of
**reserved core fields**, and MUST be emitted in a format that **supports scoped
structured context** — a [logging_context](context.md)-style capability that
attaches structured detail to every line within a scope.

The **specific wire format is not prescribed**: tailor it to the software and
its log aggregator. Two things are mandatory, nothing else:

1. **Reserved core fields** — timestamp, level, message (below).
2. **Support for scoped structured context** — rendered as reasonably
   structured, first-class data (see [Scoped structured log context](context.md)).

A format that carries the core fields but buries context inside a freeform
message tail is **non-conforming** — it leaves callers unable to attach
queryable, scope-bound detail.

# Schema (reserved core fields)

Every format MUST carry these three fields:

| Field           | Meaning                                                                                                                          |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `timestamp_utc` | When the event occurred, in UTC. Use ISO 8601 (`YYYY-MM-DDTHH:MM:SSZ`).                                                           |
| `level`         | One of `DEBUG`, `INFO`, `WARNING`, `ERROR` — the canonical set (see [Log output streams](streams.md)). Do not invent new levels. |
| `message`       | The human-readable description of the event.                                                                                     |

On top of these, a format MUST provide a structured channel for scope-bound
context fields (session/request IDs, user IDs, span IDs, error codes) — see
[Scoped structured log context](context.md).

# Formats

Both of the following are conforming **provided** they carry the reserved core
fields and support scoped structured context:

**Plain text (minimal baseline):**

```
<timestamp_utc> <level> <message> [<structured context key=value ...>]
```

```
2026-07-09T10:56:29Z INFO User 42 authenticated session_id=call-123
2026-07-09T10:56:30Z WARNING Retrying request after timeout (attempt 2/5) session_id=call-123
2026-07-09T10:57:01Z ERROR Database connection refused session_id=call-123
```

**Single-line JSON (richer option):**

```json
{"session_id": "call-123", "level": "info", "time": "2026-07-13T15:27:15.419Z", "msg": "User 42 authenticated"}
```

A single-line JSON object with context fields as **top-level keys** is the
canonical structured option: every field is queryable and alertable. Note the
context fields are written **before** the reserved keys, so a caller cannot
clobber `level` / `time` / `msg`.

Either is valid; pick what fits the application. Pure freeform text with context
lost inside `<message>` is not.

# Rationale

- **UTC, not local time.** Local time is ambiguous across servers and timezones; UTC makes logs from multiple sources comparable and trivially sortable. The field is named `timestamp_utc` to make the choice explicit.
- **Level is a first-class field.** Putting severity in a fixed position lets consumers filter (e.g. `grep ERROR`) without parsing freeform text.
- **Core fields are reserved.** Context fields MUST be written before them so a caller cannot clobber `level` / `time` / `message`.
- **Context-support is mandatory, the format is not.** Different stacks/aggregators suit different wire formats; the durable requirement is that callers can attach scope-bound structured detail. See [Scoped structured log context](context.md) for the capability and format-support contract.

# See also

- [Scoped structured log context](context.md) — the required context capability and the format-support contract.
- [Log output streams](streams.md) — which stream (`stdout`/`stderr`) each level routes to, and the alerting rationale.
- [Logging in Python](/python/logging.md) — language-specific implementation of this format plus the stream routing.
- [Scoped log context in Python](/python/logging-context.md) — the `contextvars` + `logging_context` realization and the JSON rendering.

# Citations

[1] Ingested convention directive: "All logging should be done in `<timestamp_utc> <level> <message>` format." Personal engineering convention; no external URL. Ingested 2026-07-09.
[2] Private repository `realtime-agent-server` (inaccessible) — `src/realtime_agent_server/observability/{context,logging}.py`. Established that (a) the wire format is tailorable (plain text or single-line JSON are both valid) and (b) support for scoped structured context (`logging_context`) is mandatory for any conforming format. Ingested 2026-07-23.
