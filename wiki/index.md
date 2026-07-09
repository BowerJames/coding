---
okf_version: "0.1"
---

# Coding Best Practices Wiki

> Directory listing for progressive disclosure. Maintained by the agent —
> updated whenever pages are added, moved, or removed. See `SPEC.md` for scope.
> Grouped by topic; each page's `type` is recorded in its frontmatter.

## Logging

* [Log line format](/logging/format.md) - All log output uses the `<timestamp_utc> <level> <message>` format.
* [Log output streams](/logging/streams.md) - Route DEBUG, INFO, WARNING to stdout and ERROR to stderr so error-grade output is separately alertable.

## Python

* [Logging in Python](/python/logging.md) - stdlib `logging` configured for the UTC line-format rule and DEBUG-WARNING→stdout / ERROR→stderr routing.
