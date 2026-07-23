---
type: python
title: Scoped log context in Python
description: "Implement scoped structured log context with contextvars + a logging_context context manager: caller-driven fields, copy-on-write merge, token-reset unwind, asyncio task-tree propagation."
tags: [python, logging, observability, context, asyncio]
timestamp: 2026-07-23T00:00:00Z
---

Python realizes [Scoped structured log context](/logging/context.md) with the
stdlib `contextvars` module: a process-wide `ContextVar` carries the current
scope's field bag, and a `logging_context(**fields)` context manager binds
fields for the lifetime of a block. The field set is caller-driven (not
predetermined) and every field is rendered as a top-level structured key —
here by a JSON formatter, but any formatter that renders context as first-class
data conforms.

# Mechanism

Three pieces:

```python
from contextlib import contextmanager
from contextvars import ContextVar, Token
from typing import Any

# The whole field bag for the current context. None outside any block. Always
# rebound via set() (never mutated in place) so token reset() unwinds nested
# blocks correctly — see "Merge & unwind" below.
_log_fields_var: ContextVar[dict[str, Any] | None] = ContextVar(
    "log_fields", default=None
)


def current_context_fields() -> dict[str, Any]:
    """The currently-bound context fields (a copy). Consumed by the formatter
    on each record; returns an empty dict outside any logging_context block so
    non-session log lines stay clean. A shallow copy so the caller cannot
    mutate the stored bag."""
    value = _log_fields_var.get()
    return dict(value) if value else {}


@contextmanager
def logging_context(**fields: Any):
    """Bind arbitrary structured fields to the current context for the block.
    Fields merge with the parent block; None values are kept (rendered null)."""
    parent = _log_fields_var.get()
    merged: dict[str, Any] = {**(parent or {}), **fields}
    token: Token[dict[str, Any] | None] = _log_fields_var.set(merged)
    try:
        yield
    finally:
        _log_fields_var.reset(token)
```

# Merge & unwind (copy-on-write)

`logging_context` **merges** with the parent: a child `with` block inherits the
parent's fields and adds/overrides its own. Each `set` writes a **new dict**
(`{**(parent or {}), **fields}`) — the parent dict is **never mutated in
place**. This copy-on-write discipline is load-bearing: the `Token` returned by
`set` captures the previous value, and `reset(token)` restores it. Because each
`set` stored a fresh dict, `reset` reverts to the parent's dict and the child's
fields vanish — without a trace, and without affecting the parent or any
sibling task. Mutating the parent dict in place would leak child fields past the
block boundary and defeat `reset`.

`None` passes through unchanged: a caller may explicitly bind `null`
(rendered `"key": null`), and an inner `None` overrides an outer value for the
inner block's duration; reset still reverts it on exit.

# asyncio propagation

Because every worker in this codebase is spawned via `asyncio.create_task` from
within the session coroutine, and **asyncio copies the context at task
creation**, a value bound *before* the workers start is visible to every log
call in every worker — **without threading identifiers through each call site**.

```
bind at session boundary  →  create_task copies context  →  every worker sees it
```

> **No thread pools are used here, so propagation is complete.** Were
> `loop.run_in_executor` / `ThreadPoolExecutor` introduced, a plain thread does
> **not** inherit `contextvars` — the executor callback would need
> `contextvars.copy_context().run(callable)` to preserve the bindings. Any new
> concurrency boundary must be checked.

# Rendering (the formatter)

Context fields are emitted as **top-level JSON keys, written first**, before the
reserved core fields, so a caller cannot clobber a reserved name
(`level` / `time` / `msg` / `traceback`):

```python
import json
import logging
from datetime import UTC, datetime


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload: dict[str, Any] = {}
        payload.update(current_context_fields())   # context FIRST
        payload["level"] = record.levelname.lower()
        payload["time"] = self._format_time(record)
        payload["msg"] = record.getMessage()
        if record.exc_info:
            payload["traceback"] = self.formatException(record.exc_info)
        return json.dumps(payload, default=str)    # default=str: never crash

    @staticmethod
    def _format_time(record: logging.LogRecord) -> str:
        dt = datetime.fromtimestamp(record.created, tz=UTC)
        return dt.strftime("%Y-%m-%dT%H:%M:%S") + f".{int(record.msecs):03d}Z"
```

- `default=str` so non-serialisable values (a `datetime`, a custom object)
  stringify instead of crashing the log call.
- `None` renders as `null`.
- Outside any context block, `current_context_fields()` returns `{}`, so
  startup / health-check / framework log lines stay clean.

> **JSON is one valid rendering, not the standard.** The context mechanism is
> format-independent (see [Log line format](/logging/format.md)): any formatter
> that renders context as first-class structured data, written before the
> reserved fields, conforms. Plain text with structured `key=value` pairs is
> equally conforming.

# Usage

```python
with logging_context(session_id="call-123", caller_no="07123456789"):
    # every log line carries session_id + caller_no
    with logging_context(hook="property_maintenance"):
        # also carries hook
        ...
    # hook is gone again; session_id/caller_no retained
```

# Notes

- **Why `contextvars`** — it is the stdlib primitive for per-context state that
  propagates across `asyncio` tasks (and threads, with the caveat above). It is
  the right tool here, not `threading.local` (which is blind to asyncio and
  would not propagate across `create_task`).
- **Per-request isolation** — each session binds its own fields at its own
  boundary; because context is copied per task and reset on block exit, nothing
  leaks from one connection into the next.
- **Idempotent `configure_logging`** — clears existing handlers on each call, so
  it is safe to invoke repeatedly (e.g. from tests).

# See also

- [Scoped structured log context](/logging/context.md) — the language-agnostic standard this realizes.
- [Log line format](/logging/format.md) — the format contract (tailorable, but core fields + context-support mandatory).
- [Logging in Python](/python/logging.md) — the baseline core-fields formatter + stream routing this layer sits on top of.
- [Log output streams](/logging/streams.md) — routing is orthogonal; context fields ride the stream the record's level dictates.

# Citations

[1] Private repository `realtime-agent-server` (inaccessible):

- `src/realtime_agent_server/observability/context.py` — the `logging_context` context manager, `current_context_fields()` reader, merge-on-nest copy-on-write unwind, and the asyncio-propagation analysis (including the `run_in_executor` / `copy_context().run` caveat).
- `src/realtime_agent_server/observability/logging.py` — the `JsonFormatter` (context-first rendering, `default=str`) and the two-handler stream routing.

Ingested 2026-07-23.
