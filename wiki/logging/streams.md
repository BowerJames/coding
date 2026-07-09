---
type: practice
title: Log output streams
description: "Route DEBUG, INFO, WARNING to stdout and ERROR to stderr so error-grade output is separately alertable."
tags: [logging, observability, alerting]
timestamp: 2026-07-09T14:31:09Z
---

Every log line emitted by code in this wiki's scope is routed to one of the two
OS standard streams based on its severity. The rule is a **clean partition**:
`stdout` carries normal-flow output and is never alertable; `stderr` carries
error-grade output and is the alertable channel.

```
DEBUG    -> stdout
INFO     -> stdout
WARNING  -> stdout
ERROR    -> stderr
```

# Schema

| Level    | Stream | Role                                                                              |
| -------- | ------ | --------------------------------------------------------------------------------- |
| `DEBUG`  | stdout | Diagnostic detail; normal observability. Never alertable.                         |
| `INFO`   | stdout | Normal operation. Never alertable.                                                |
| `WARNING`| stdout | Recoverable / soft-fail conditions worth reviewing. Not alertable.               |
| `ERROR`  | stderr | Failures / exceptional conditions. **The alertable channel.**                     |

These four are the **canonical level set**. Do not add more, and do not *emit*
severities above `ERROR` from your own code (`CRITICAL` was dropped from the
canonical set on 2026-07-09). See [Log line format](format.md) for the line
structure.

If a *dependency* or framework emits severities above `ERROR` anyway (e.g.
`CRITICAL`/`FATAL`), route them to **stderr alongside `ERROR`** — stderr is the
single home for error-grade output.

# Rationale

- **`ERROR` is the alertable signal.** Routing `ERROR` to its own stream lets
  alerting / monitoring pipelines watch `stderr` in isolation and fire on
  error-grade output without wading through routine `DEBUG`/`INFO`/`WARNING`
  noise. A monitor can tail `stderr` and treat *every* line as an alert
  candidate — no thresholds, no parsing, no filtering.
- **Keep `stdout` clean of error-grade output.** `stdout` is the "normal flow"
  stream: tail it, pipe it, archive it, or feed it to dashboards without ever
  risking an alert. That invariant holds only if `ERROR` never leaks onto
  `stdout`, which is why `ERROR` is **not** mirrored to both streams.
- **No duplication.** `ERROR` appears exactly once, on `stderr`. Mirroring it to
  `stdout` would double-count errors in any setup that captures both streams
  (e.g. `prog > out.log 2> err.log`), break dedup, and re-introduce the very
  noise this routing exists to avoid. If a deployment only collects `stdout` and
  therefore misses errors, fix the **collection layer** (collect `stderr` too) —
  do not duplicate at the application layer.
- **Standard streams, not bespoke files.** Using the OS standard streams keeps
  the rule portable, pipe-friendly, and free of config / environment coupling.

# Examples

```sh
# Show only the alertable channel (ERROR) in isolation:
./myprogram 1>/dev/null

# Show only normal-flow output (DEBUG, INFO, WARNING), suppressing errors:
./myprogram 2>/dev/null

# Capture them separately — ERROR lands in err.log exactly once:
./myprogram 1>normal.log 2>err.log
```

# See also

- [Log line format](format.md) — the `<timestamp_utc> <level> <message>` structure every line must take.
- [Logging in Python](/python/logging.md) — the language-specific implementation of this routing with stdlib `logging`.
- [Log vs. raise](/error-handling/log-vs-raise.md) — how this `ERROR`/`WARNING` distinction applies to tolerated data-model anomalies (log at `ERROR`, don't raise).
- [Fault tolerance](/error-handling/fault-tolerance.md) — the error-handling policy that consumes this routing: survive by default, fail narrowly, and what an `ERROR` means (a broken assumption).

# Citations

[1] Ingested convention directive: "DEBUG → stdout, INFO → stdout, WARNING → stdout, ERROR → stderr; ERROR must be distinguishable so alerts can be used with them. ERROR routes to stderr only (no mirroring to stdout)." Personal engineering convention; no external URL. Ingested 2026-07-09.
