---
type: practice
title: "Log vs. raise — absorb recoverable anomalies, propagate the rest (always log ERROR)"
description: "For a given anomaly, absorb it (recover + log ERROR at the recovery site) when you can still satisfy the caller, and propagate it (let it bubble to a handler) when you can't. Log where handled — exactly once, at the handler. Whether the process survives is a separate decision — see Fault tolerance."
tags: [error-handling, logging, observability, soft-assertions, error-propagation]
timestamp: 2026-07-10T16:20:00Z
---

> **Superseded thesis (2026-07-09).** An earlier version of this page said
> internal-invariant violations should *"crash the process"* ("crash on bugs you
> own"). That stance is **superseded by [Fault tolerance](fault-tolerance.md)**:
> internal bugs fail the *request* (e.g. `HTTP 500`), not the process — crashing
> is a last resort only. This page now covers a narrower question: **absorb vs
> propagate**. The crash-vs-survive decision lives in
> [Fault tolerance](fault-tolerance.md).

> **Refined (2026-07-10).** The logging rule is sharpened from *"log `ERROR` at
> the absorb/propagate site"* to **"log where handled"**: absorbed failures are
> logged at the recovery site (the only frame that sees them); propagated
> failures are logged **once, at the handler**, not at every intermediate frame
> (which only double-logs and drops the trace). The invariant "every failure is
> logged `ERROR`" still holds — it now means *exactly once, at the handler*.
> Source: [Grinberg](#citations) [5].

# The principle

When an unexpected condition violates your data model, the immediate decision is
**do you *absorb* it or *propagate* it?** The logging rule that ties them
together is **log where the error is *handled***, not where it is merely caught
and passed on.

- **Absorb** — recover and continue: serve a fallback, use a default, or carry
  on with the parts that are still valid. Choose this when you can still satisfy
  the caller despite the anomaly. **Log `ERROR` here** — this frame is the only
  place that ever sees the error (it never reaches the boundary), so the log is
  mandatory.
- **Propagate** — surface the failure: throw, return `Result::Err`, or let it
  bubble. Choose this when you cannot satisfy the caller. **Do not log here** —
  the eventual *handler* logs once (see below). Logging at every intermediate
  frame produces N context-poor records for one error.

**"Handled" ≠ "caught".** Handling means a real recovery *decision* — you
recovered (absorb), or you turned the failure into a clean boundary outcome
(the request returns `500`, the process exits cleanly). Merely catching to
re-raise or translate is *propagation*, and logging there is the
**log-and-re-raise smell**: it duplicates the record the handler will emit, and
the hand-rolled line almost always omits the stack trace. Reach for it only as a
deliberate breadcrumb at a specific layer, never as the default.

The one thing you never do is **silently swallow** the anomaly. Absorbing
*without* logging is the **"silent failure"** anti-pattern: bad data leaves no
trail and the bug becomes invisible. Logging at `ERROR` (the alertable channel;
see [Log output streams](/logging/streams.md)) is what makes an absorbed anomaly
safe.

> **Absorb vs propagate is the [error taxonomy](error-taxonomy.md) in two
> rows.** Absorb = T1+T2 (you recovered); propagate = T3+T4 (you let it bubble
> to a handler). The taxonomy refines this page; this page owns the *logging*
> discipline (never silent; log where handled).

This page answers *absorb vs propagate* and *where the log goes*. What happens
to the **process** when you propagate (does the request fail, or does the whole
process crash?) is decided by [Fault tolerance](fault-tolerance.md): the default
is to **fail the request, not the process**.

# Schema

| Can you still satisfy the caller? | Action | Log? |
| --------------------------------- | ------ | ---- |
| Yes — via the normal path or a reasonable fallback | **Absorb**: serve the result/fallback and continue. | **`ERROR`, here** (recovery site). |
| No — the operation genuinely cannot proceed | **Propagate**: surface the error to the caller (`throw` / `Result::Err` / let it bubble / `HTTP 400`–`500` *at the boundary*). | **Not here** — the handler logs once. |

Every failure is logged `ERROR` **exactly once**, at the handler. Absorb logs
at the recovery site because no boundary ever sees it; propagate logs once at
the handler that finally absorbs it or cleanly fails the request. Neither row
silently swallows.

# Rationale

- **Absorb when you can, propagate when you can't.** Absorbing keeps the service
  resilient to the world's messiness — a malformed record shouldn't sink a whole
  batch import. Propagating keeps failure honest when the operation truly can't
  be completed — a missing required field can't be invented.
- **Log `ERROR` exactly once, at the handler.** An absorbed anomaly is still a
  failure — an assumption broke (see [Fault tolerance](fault-tolerance.md)) — and
  you want to be alerted, so log it at the recovery site (the only frame that
  sees it). A propagated failure is logged once at the handler that finally deals
  with it, with a stack trace — never re-logged at every frame.
- **Never swallow silently.** The discipline that separates healthy fault
  tolerance from the silent-failure anti-pattern is the log line. An absorb
  without an `ERROR` log is a bug.
- **Avoid log-and-re-raise.** Catching, logging, and re-raising usually
  double-logs (your line plus the handler's) and drops the trace. Propagate
  without logging; let the handler log once.
- **Postel's Law, boundary-scoped.** "Be liberal in what you accept" justifies
  *absorbing* deviations in external data. Apply it at your system's boundaries,
  never to your internal invariants (absorbing an internal violation without
  flagging it masks real bugs).

# Examples

**Absorb — log ERROR at the recovery site, continue.** A batch importer reads
records; one carries an unexpected extra field and a field whose type changed.
The known fields still yield a valid result:

```
2026-07-09T13:42:00Z ERROR record 42: field "status" had unexpected shape "number" (expected string); imported with default
```

Don't propagate — the import still produces useful output, and you now know
which producer is drifting. The log belongs *here* because no boundary will
ever see this recovered error.

**Propagate — surface to caller; the handler logs once.** A request omits a
required field, or an internal state-machine assumption fails mid-request. You
cannot fulfil it, so let it propagate **without logging**; the boundary handler
logs `ERROR` (with a stack trace) and returns `400`/`500` (or the process exits
cleanly). The process keeps running for the next request.

- If a framework already supplies that boundary (Flask's
  `full_dispatch_request`, an Express error middleware), your handler should be
  **T4** — do nothing — rather than re-implementing the log / `500` / rollback
  per endpoint (see [Error handling in Python](/python/error-handling.md)).
- If there is no framework, your own top-level wrapper *is* the handler and logs
  there (see [Fault tolerance](fault-tolerance.md)).

*(An earlier version of this example logged at the propagate site and raised the
internal case to crash the process; both are superseded — log where handled, and
fail the request, not the process.)*

# See also

- [Error taxonomy](error-taxonomy.md) — the 2×2 this absorb/propagate split summarises (absorb = T1+T2, propagate = T3+T4).
- [EAFP vs LBYL](eafp-vs-lbyl.md) — the *catching-style* question, which is language-dependent.
- [Fault tolerance](fault-tolerance.md) — the policy this page defers to: survive by default, fail narrowly (fallback → fail request → crash as last resort); what an `ERROR` means.
- [Log output streams](/logging/streams.md) — why `ERROR` is the alertable channel failures are logged on.
- [Log line format](/logging/format.md) — the `<timestamp_utc> <level> <message>` shape the example log line follows.
- [Logging in Python](/python/logging.md) — language-specific implementation of the routing.
- [Error handling in Python](/python/error-handling.md) — the Python realization; `logger.exception()` logs the trace at the handler.

# Citations

[1] Synthesised from user query (2026-07-09): "unexpected error behaviours that don't break the application but go against its data model should be logged as errors but not raised." Filed back as a practice.
[2] Ingested convention directive (2026-07-09): "Support fault-tolerant applications; all failures logged as errors; strive not to crash; reasonable fallbacks (REST returns `500`, not a crash)." Supersedes the original "crash on bugs you own" thesis; crash-vs-survive now lives in [Fault tolerance](fault-tolerance.md).
[3] Postel's Law (the Robustness Principle): "be conservative in what you send, be liberal in what you accept." Cited for its *boundary* form — absorbing deviations in external data.
[4] Soft assertions — a testing idiom (e.g. TestNG) in which failures are recorded and the run continues; the source of the "record, don't abort" technique adapted here as "absorb, but log."
[5] Miguel Grinberg, "The Ultimate Guide to Error Handling in Python" (2024-10-07), https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python — source of the "log where handled / log once at the handler" refinement and the log-and-re-raise smell; the absorb/propagate split is generalised into the [Error taxonomy](error-taxonomy.md).
