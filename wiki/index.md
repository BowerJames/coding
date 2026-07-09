---
okf_version: "0.1"
---

# Coding Best Practices Wiki

> Directory listing for progressive disclosure. Maintained by the agent —
> updated whenever pages are added, moved, or removed. See [SPEC.md](SPEC.md) for scope.
> Grouped by topic; each page's `type` is recorded in its frontmatter.

## Logging

* [Log line format](/logging/format.md) - All log output uses the `<timestamp_utc> <level> <message>` format.
* [Log output streams](/logging/streams.md) - Route DEBUG, INFO, WARNING to stdout and ERROR to stderr so error-grade output is separately alertable.

## Error handling

* [Fault tolerance](/error-handling/fault-tolerance.md) - Survive by default: fail at the narrowest scope (fallback → fail the request → crash only as a last resort); every failure logged at `ERROR`. An error means an application assumption broke (external dependency, data model, or internal state).
* [Log vs. raise](/error-handling/log-vs-raise.md) - Absorb recoverable anomalies (log + continue/fallback), propagate the rest (throw/`Err`/boundary); always log `ERROR`. Crash-vs-survive is decided by Fault tolerance.

## Workflow

* [Task running with justfile](/workflow/justfile.md) - Every project uses just as the single entry point for tasks/commands/scripts; recipes wrap bare tool commands or native-language scripts; dotenv always on; same recipes run locally and in CI.

## Python

* [Logging in Python](/python/logging.md) - stdlib `logging` configured for the UTC line-format rule and DEBUG-WARNING→stdout / ERROR→stderr routing.
* [Type safety in Python](/python/type-safety.md) - Run mypy in strict mode; annotate every function; avoid `Any` by receiving unknown data as `object` and narrowing (prefer dataclasses / pydantic as containers); justify any `type: ignore`.
* [Linting and formatting in Python](/python/linting.md) - Use ruff for both linting (`ruff check`) and formatting (`ruff format`); defaults are the floor, run locally and in CI. Ruff is not a type checker — that stays mypy's job.
* [Streaming seams in Python](/python/streaming.md) - Implement the stream-seam pattern with `asyncio.Queue`: a single-pass async-iterator `Stream` and a `push`/`end` `StreamWriter` linked by a shared `_Core`, paired by `create_stream()` as a frozen `StreamWiring` dataclass (`.producer`/`.consumer`). Modern typing (PEP 695 generics, `Self`, `__slots__`, `@dataclass`).

## Patterns

* [Stream seam](/patterns/streaming.md) - Split an async stream into a consumer iterator and a producer writer sharing one queue, paired by a `create_stream()` factory — so a provider can return the read side while filling it from a background task. Single-pass, single-consumer, sentinel-terminated, idempotent close.
