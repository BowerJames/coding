---
type: python
title: Logging in Python
description: "Implement the log-format and stream-routing conventions with stdlib logging: a stdout handler (DEBUG-WARNING) and a stderr handler (ERROR+)."
tags: [python, logging, observability]
timestamp: 2026-07-23T00:00:00Z
---

Python's standard `logging` module implements the wiki's logging conventions —
the [line format](/logging/format.md) and the [output-stream routing](/logging/streams.md)
— via **two `StreamHandler`s attached to the root logger**, one per stream,
plus a level filter to keep `ERROR` off `stdout`.

# The routing, concretely

- **`stdout` handler** — emits `DEBUG`, `INFO`, `WARNING`. Configured with
  `level=DEBUG` *and* a filter that caps it at `WARNING`, so `ERROR` is excluded.
- **`stderr` handler** — emits `ERROR` (and any higher severity). This is the
  alertable channel; see `/logging/streams.md`.

# Setup

```python
import logging
import sys
import time


class _MaxLevelFilter(logging.Filter):
    """Keep only records at or below a given level."""

    def __init__(self, max_level: int) -> None:
        super().__init__()
        self.max_level = max_level

    def filter(self, record: logging.LogRecord) -> bool:
        return record.levelno <= self.max_level


def configure_logging() -> None:
    # <timestamp_utc> <level> <message>  — see /logging/format.md
    formatter = logging.Formatter(
        fmt="%(asctime)s %(levelname)s %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%SZ",
    )
    formatter.converter = time.gmtime  # force UTC (asctime defaults to local)

    # stdout: DEBUG, INFO, WARNING  (ERROR excluded via the filter)
    stdout_handler = logging.StreamHandler(sys.stdout)
    stdout_handler.setLevel(logging.DEBUG)
    stdout_handler.addFilter(_MaxLevelFilter(logging.WARNING))
    stdout_handler.setFormatter(formatter)

    # stderr: ERROR and above  (the alertable channel)
    stderr_handler = logging.StreamHandler(sys.stderr)
    stderr_handler.setLevel(logging.ERROR)
    stderr_handler.setFormatter(formatter)

    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.addHandler(stdout_handler)
    root.addHandler(stderr_handler)
```

# Usage

```python
configure_logging()
log = logging.getLogger(__name__)
log.debug("cache warmed in %.2fs", 0.31)      # -> stdout
log.info("user %s authenticated", 42)         # -> stdout
log.warning("retrying after timeout (2/5)")   # -> stdout
log.error("database connection refused")      # -> stderr  (alertable)
```

# Notes

- **Why the filter is required.** A handler's `level` is a *minimum* (≥)
  threshold, not an exact match. So a bare `DEBUG`-level `stdout` handler would
  also emit `ERROR`. `_MaxLevelFilter(logging.WARNING)` caps `stdout` at
  `WARNING`, leaving `ERROR` to be emitted by the `stderr` handler only.
  Without it, `ERROR` would appear on **both** streams — polluting `stdout` and
  defeating the alerting rationale (see `/logging/streams.md`).
- **`formatter.converter = time.gmtime`.** `%(asctime)s` defaults to *local*
  time. Setting `converter` to `time.gmtime` honours the UTC requirement of the
  line-format rule. Forgetting this is the most common format bug.
- **`basicConfig` cannot split streams.** `logging.basicConfig()` configures a
  *single* handler (to `sys.stderr` by default) and has no way to route records
  to different streams by level. In particular, `basicConfig(stream=sys.stdout)`
  sends *everything* to `stdout` — that is the common wrong turn. Use explicit
  handlers as shown above.
- **`CRITICAL` is out of scope.** The canonical level set is four levels (see
  `/logging/streams.md`). The `stderr` handler's `level=ERROR` therefore emits
  `ERROR` only in practice; if a library logs `CRITICAL`, it lands on `stderr`
  alongside `ERROR`, which is consistent with the rule.
- **This is the core-fields baseline; add the context layer.** The plain `%(asctime)s %(levelname)s %(message)s` formatter above carries the reserved core fields only. To satisfy the scoped-context requirement of [Log line format](/logging/format.md) — attaching structured per-scope detail without per-call-site plumbing — pair this setup with the [contextvars + `logging_context`](/python/logging-context.md) layer (which also shows the JSON rendering that carries context fields as top-level keys).
- **Prefer `logger.exception()` inside an `except` block.** When you log at an
  error handler, `logger.exception(msg)` emits the message **and the current
  exception's stack trace** (captured automatically from `sys.exc_info()`), which
  is the single most useful artifact for later debugging. `logger.error(str(e))`
  drops the trace. This is the Python mechanism for the
  [error-handling](/python/error-handling.md) "log where handled" rule — log
  once, at the handler, *with* the trace. Logging anywhere short of the handler — `logger.error(str(e))` before a `raise` — is the [log-and-re-raise](/error-handling/log-and-re-raise.md) anti-pattern: double-logged, and traceless.

# See also

- [Log line format](/logging/format.md) — the `<timestamp_utc> <level> <message>` structure this formatter produces.
- [Scoped log context in Python](/python/logging-context.md) — the `contextvars` + `logging_context` layer that adds scoped structured context on top of this baseline formatter.
- [Log output streams](/logging/streams.md) — the level→stream rule this setup implements.
- [Error handling in Python](/python/error-handling.md) — `logger.exception()` logs the trace at the error handler; log where handled.
- [Log and re-raise](/error-handling/log-and-re-raise.md) — the anti-pattern this `logger.exception()`-vs-`logger.error()` note prevents: traceless logging at a re-raise site.

# Citations

[1] Derives from the same ingested convention directive as [/logging/streams.md](/logging/streams.md): DEBUG/INFO/WARNING → stdout, ERROR → stderr (stderr only). Personal engineering convention; no external URL. Ingested 2026-07-09.
[2] Miguel Grinberg, "The Ultimate Guide to Error Handling in Python" (2024-10-07), https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python — `logger.exception()` over `logger.error()` for stack traces (noted in the article and its comment thread).
