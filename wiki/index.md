---
okf_version: "0.1"
---

# Coding Best Practices Wiki

> Directory listing for progressive disclosure. Maintained by the agent —
> updated whenever pages are added, moved, or removed. See [SPEC.md](SPEC.md) for scope.
> Grouped by topic; each page's `type` is recorded in its frontmatter.

## Logging

* [Log line format](/logging/format.md) - Every log line carries reserved core fields (UTC timestamp, level, message) and supports scoped structured context; the wire format is tailorable (plain text or JSON).
* [Scoped structured log context](/logging/context.md) - Provide a logging_context capability that binds structured fields to the current scope, propagated automatically by the runtime, so every log line under that scope carries them with no per-call-site plumbing.
* [Log output streams](/logging/streams.md) - Route DEBUG, INFO, WARNING to stdout and ERROR to stderr so error-grade output is separately alertable.

## Error handling

* [Error taxonomy](/error-handling/error-taxonomy.md) - Classify any error by origin (new vs bubbled-up) and recoverability → four handling strategies. Most code should do nothing and let non-recoverable errors flow to a single handler.
* [EAFP vs LBYL](/error-handling/eafp-vs-lbyl.md) - Look Before You Leap vs Easier to Ask Forgiveness than Permission. Exception languages favour EAFP; Rust/Go make explicit checking idiomatic. Let errors flow where the language allows.
* [Fault tolerance](/error-handling/fault-tolerance.md) - Survive by default: fail at the narrowest scope (fallback → fail the request → crash only as a last resort); every failure logged at `ERROR`. An error means an application assumption broke (external dependency, data model, or internal state).
* [Log vs. raise](/error-handling/log-vs-raise.md) - Absorb recoverable anomalies (log at the recovery site), propagate the rest (the handler logs once). Always log `ERROR` — exactly once, where handled.
* [Log and re-raise](/error-handling/log-and-re-raise.md) - Catching an error, logging it, then re-raising: double-logs and drops the stack trace. Log where handled — once, at the handler; propagate silently between frames.

## Workflow

* [Task running with justfile](/workflow/justfile.md) - Every project uses just as the single entry point for tasks/commands/scripts; recipes wrap bare tool commands or native-language scripts; dotenv always on; same recipes run locally and in CI.

## Python

* [Error handling in Python](/python/error-handling.md) - Python realization of the error taxonomy: try/except, EAFP, custom exceptions, `logger.exception()`, no `assert`, one top-level `except Exception` catch-all, let the framework (Flask/Tkinter) do it.
* [Logging in Python](/python/logging.md) - stdlib `logging` configured for the UTC line-format rule and DEBUG-WARNING→stdout / ERROR→stderr routing.
* [Scoped log context in Python](/python/logging-context.md) - Realize scoped structured log context with `contextvars` + a `logging_context` context manager: caller-driven fields, copy-on-write merge, token-reset unwind, asyncio task-tree propagation.
* [Type safety in Python](/python/type-safety.md) - Run mypy in strict mode; annotate every function; avoid `Any` by receiving unknown data as `object` and narrowing (prefer dataclasses / pydantic as containers); justify any `type: ignore`.
* [Linting and formatting in Python](/python/linting.md) - Use ruff for both linting (`ruff check`) and formatting (`ruff format`); defaults are the floor, run locally and in CI. Ruff is not a type checker — that stays mypy's job.
* [Streaming seams in Python](/python/streaming.md) - Implement the stream-seam pattern with `asyncio.Queue`: a single-pass async-iterator `Stream` and a `push`/`end` `StreamWriter` linked by a shared `_Core`, paired by `create_stream()` as a frozen `StreamWiring` dataclass (`.producer`/`.consumer`). Modern typing (PEP 695 generics, `Self`, `__slots__`, `@dataclass`).

## Patterns

* [Stream seam](/patterns/streaming.md) - Split an async stream into a consumer iterator and a producer writer sharing one queue, paired by a `create_stream()` factory — so a provider can return the read side while filling it from a background task. Single-pass, single-consumer, sentinel-terminated, idempotent close.
