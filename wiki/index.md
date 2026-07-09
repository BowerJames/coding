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

## Python

* [Logging in Python](/python/logging.md) - stdlib `logging` configured for the UTC line-format rule and DEBUG-WARNING→stdout / ERROR→stderr routing.
* [Type safety in Python](/python/type-safety.md) - Run mypy in strict mode; annotate every function; avoid `Any` by receiving unknown data as `object` and narrowing (prefer dataclasses / pydantic as containers); justify any `type: ignore`.
