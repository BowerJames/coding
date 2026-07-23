---
type: practice
title: Scoped structured log context
description: "Provide a logging_context capability that binds arbitrary structured fields to the current scope, propagated automatically by the runtime, so every log line under that scope carries them with zero per-call-site plumbing."
tags: [logging, observability, context]
timestamp: 2026-07-23T00:00:00Z
---

A log line should carry not just its fixed core fields — timestamp, level,
message (see [Log line format](format.md)) — but also whatever structured
detail the *current scope* knows: a session/request ID, a user ID, a tenant, a
hook name, a correlation/span ID. **Scope-bound context** is the standard way to
get that detail onto every log line emitted within a scope **without threading
identifiers through every call site.**

The standard is to **supply a `logging_context`-style capability** — a scoped
binding that attaches arbitrary structured fields to the current execution
context for the lifetime of a block. This is not an optional nicety: any
conforming log format ([Log line format](format.md)) **must** support it.

> **Required, not optional.** A format that carries only the fixed core fields
> and buries context inside a freeform message tail is non-conforming — it
> leaves callers unable to attach queryable, scope-bound detail. See
> [Log line format](format.md) for the format contract.

# The capability contract

A `logging_context` capability MUST provide:

- **Caller-driven field set.** The schema is **not** predetermined by the
  logging layer. Callers pass arbitrary `key=value` pairs and every field is
  emitted. No central registry can anticipate every scope's context; the scope
  itself knows what it carries.
- **Scoped.** Fields are bound for the lifetime of a block (a context manager
  or language equivalent) and unbound on exit. Bind at the top of a
  per-connection / per-request / per-unit-of-work scope.
- **Merge-on-nesting.** A child scope **inherits** the parent's fields and
  adds/overrides its own. On exit the child's fields revert; the parent is
  unaffected.
- **Clean unwind, no leaks.** Child fields do not leak to sibling scopes or to
  the next request. Resetting a child block restores the parent's context
  exactly — the basis of correct per-request isolation.
- **Automatic runtime propagation.** Fields propagate across the runtime's
  concurrency unit (an async task tree, a goroutine, a thread) **without the
  caller threading identifiers into each function call**. This is the central
  payoff: bind once at the scope boundary, and every nested call site — and
  every background task spawned within — inherits the bindings.
- **`None` / null passthrough.** A caller may explicitly bind a null value;
  it is rendered as `null`, not dropped.

> **Propagation is a runtime property — verify it.** The capability piggybacks
> on the language's context primitive (`contextvars`, `context.Context`,
> `AsyncLocalStorage`, …), and that primitive propagates *only where the
> runtime copies context*. Asynchronous task spawn usually copies it; a bare
> thread or a thread-pool executor usually does **not**. Where the boundary
> exists, the caller must copy the context across it explicitly (e.g.
> `contextvars.copy_context().run(...)`). If you introduce a new concurrency
> boundary, confirm context still propagates.

# The format-support contract

For a format to support this capability, it MUST:

- **Render context fields as reasonably-structured, first-class data** —
  queryable keys a log aggregator can filter and alert on — not as prose buried
  in the message tail. (Plain text can satisfy this with structured `key=value`
  pairs or a trailing JSON blob; a single-line JSON object with context fields
  as top-level keys is the canonical structured option — see
  [Log line format](format.md).)
- **Emit context fields before reserved core fields**, so a caller passing a
  reserved name (timestamp / level / message) cannot clobber it.
- **Tolerate non-serialisable values** (stringify them; never crash a log call
  because a field value is a `datetime` or an object).
- **Emit nothing extra outside any scope.** When no context is bound (startup,
  health checks, framework loggers), non-session log lines stay clean.

# Why this over alternatives

- **Over threading IDs through call sites:** every function would need a
  `session_id=` / `request_id=` parameter (and every intermediate frame), which
  is noisy, easy to forget, and viral through the signature. Scoped context
  binds once and is invisible to intermediate code.
- **Over a global/static context:** a global cannot isolate concurrent
  requests — its fields would leak across sessions. Scope-bound context
  isolates per execution context.
- **Over embedding context in the message string:** freeform text is not
  queryable; you cannot alert on `user_id=42` reliably. First-class fields are.

# In other languages

The mechanism is universal; the primitive differs:

- **Python** — `contextvars.ContextVar` + a `logging_context` context manager.
  See [Scoped log context in Python](/python/logging-context.md).
- **Rust** — `tracing` spans (`#[instrument]` / `tracing::span!`) carry
  structured fields across an `await` tree.
- **Go** — `context.Context` for propagation, paired with `slog`'s structured
  handlers for emission.
- **TypeScript / Node** — `AsyncLocalStorage` provides the async-context slot.

# See also

- [Log line format](format.md) — the format contract: tailorable wire format, but the reserved core fields + support for this context capability are mandatory.
- [Log output streams](streams.md) — routing is orthogonal to context; context fields ride whatever stream the record's level dictates.
- [Scoped log context in Python](/python/logging-context.md) — the worked realization (`contextvars` + `logging_context`).

# Citations

[1] Private repository `realtime-agent-server` (inaccessible) — `src/realtime_agent_server/observability/context.py`. Provides the `logging_context` context manager, `current_context_fields()` reader, merge-on-nest with copy-on-write unwind, and the asyncio-propagation analysis. Ingested 2026-07-23.
