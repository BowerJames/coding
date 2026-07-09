# Coding Best Practices Wiki

A personal, LLM-readable knowledge base of general software-engineering best
practices, authored in the
[Open Knowledge Format (OKF)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).

## What's here

- **`wiki/SPEC.md`** — the wiki's purpose, scope, page types, and conventions. Start here.
- **`wiki/index.md`** — a directory listing of all pages (progressive disclosure).
- **`wiki/log.md`** — a newest-first change history.
- **`wiki/practices/`, `wiki/patterns/`, `wiki/concepts/`, `wiki/sources/`** —
  the knowledge itself, added over time.

## How it's maintained

The wiki is maintained by an AI coding agent (in the
[pi](https://github.com/earendil-works/pi-coding-agent) harness) under three
operations:

1. **Ingest** — provide a source; the agent reads it, summarises it, and updates
   the wiki + index + log.
2. **Query** — ask a question; the agent synthesises a cited answer from the
   wiki and files valuable answers back in.
3. **Lint** — the agent health-checks for contradictions, stale claims, orphan
   pages, and missing concepts.

See `wiki/SPEC.md` for the full conventions.
