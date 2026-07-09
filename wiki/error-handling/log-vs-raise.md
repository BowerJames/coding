---
type: practice
title: "Log vs. raise — absorb recoverable anomalies, propagate the rest (always log ERROR)"
description: "For a given anomaly, absorb it (log + continue/serve a fallback) when you can still satisfy the caller, and propagate it (throw / return Err / surface at the boundary) when you can't. Always log ERROR. Whether the process survives or crashes is a separate decision — see Fault tolerance."
tags: [error-handling, logging, observability, soft-assertions, error-propagation]
timestamp: 2026-07-09T14:31:09Z
---

> **Superseded thesis (2026-07-09).** An earlier version of this page said
> internal-invariant violations should *"crash the process"* ("crash on bugs you
> own"). That stance is **superseded by [Fault tolerance](fault-tolerance.md)**:
> internal bugs fail the *request* (e.g. `HTTP 500`), not the process — crashing
> is a last resort only. This page now covers a narrower question: **absorb vs
> propagate**. The crash-vs-survive decision lives in
> [Fault tolerance](fault-tolerance.md).

# The principle

When an unexpected condition violates your data model, the immediate decision is
**do you *absorb* it or *propagate* it?** — and either way you **log `ERROR`**.

- **Absorb** — log `ERROR` and continue: serve a fallback, use a default, or
  carry on with the parts that are still valid. Choose this when you can still
  satisfy the caller despite the anomaly.
- **Propagate** — log `ERROR` and surface the failure: throw, return
  `Result::Err`, or raise *at the request boundary*. Choose this when you cannot
  satisfy the caller.

The one thing you never do is **silently swallow** the anomaly. Absorbing
*without* logging is the **"silent failure"** anti-pattern: bad data leaves no
trail and the bug becomes invisible. Logging at `ERROR` (the alertable channel;
see [Log output streams](/logging/streams.md)) is what makes an absorbed anomaly
safe.

This page answers *absorb vs propagate*. What happens to the **process** when
you propagate (does the request fail, or does the whole process crash?) is
decided by [Fault tolerance](fault-tolerance.md): the default is to **fail the
request, not the process**.

# Schema

| Can you still satisfy the caller? | Action |
| --------------------------------- | ------ |
| Yes — via the normal path or a reasonable fallback | **Absorb**: log `ERROR`, serve the result/fallback. |
| No — the operation genuinely cannot proceed | **Propagate**: log `ERROR`, surface the error to the caller (`throw` / `Result::Err` / boundary raise / `HTTP 400`–`500`). |

Both rows log `ERROR`. Neither row silently swallows.

# Rationale

- **Absorb when you can, propagate when you can't.** Absorbing keeps the service
  resilient to the world's messiness — a malformed record shouldn't sink a whole
  batch import. Propagating keeps failure honest when the operation truly can't
  be completed — a missing required field can't be invented.
- **Always log `ERROR`.** An absorbed anomaly is still a failure — an assumption
  broke (see [Fault tolerance](fault-tolerance.md)) — and you want to be alerted.
  The fallback hides the symptom *from the caller*, never from *the operator*.
- **Never swallow silently.** The discipline that separates healthy fault
  tolerance from the silent-failure anti-pattern is the log line. An absorb
  without an `ERROR` log is a bug.
- **Postel's Law, boundary-scoped.** "Be liberal in what you accept" justifies
  *absorbing* deviations in external data. Apply it at your system's boundaries,
  never to your internal invariants (absorbing an internal violation without
  flagging it masks real bugs).

# Examples

**Absorb — log ERROR, continue.** A batch importer reads records; one carries an
unexpected extra field and a field whose type changed. The known fields still
yield a valid result:

```
2026-07-09T13:42:00Z ERROR record 42: field "status" had unexpected shape "number" (expected string); imported with default
```

Don't propagate — the import still produces useful output, and you now know
which producer is drifting.

**Propagate — log ERROR, surface to caller, process survives.** A request omits
a required field, or an internal state-machine assumption fails mid-request. You
cannot fulfil it → log `ERROR`, return `400`/`500` (or `Result::Err` / raise *at
the request boundary*). The process keeps running for the next request. *(An
earlier version of this example raised the internal case to crash the process;
that is superseded — see [Fault tolerance](fault-tolerance.md).)*

# See also

- [Fault tolerance](fault-tolerance.md) — the policy this page defers to: survive by default, fail narrowly (fallback → fail request → crash as last resort); what an `ERROR` means.
- [Log output streams](/logging/streams.md) — why `ERROR` is the alertable channel absorbed/propagated failures are logged on.
- [Log line format](/logging/format.md) — the `<timestamp_utc> <level> <message>` shape the example log line follows.
- [Logging in Python](/python/logging.md) — language-specific implementation of the routing.

# Citations

[1] Synthesised from user query (2026-07-09): "unexpected error behaviours that don't break the application but go against its data model should be logged as errors but not raised." Filed back as a practice.
[2] Ingested convention directive (2026-07-09): "Support fault-tolerant applications; all failures logged as errors; strive not to crash; reasonable fallbacks (REST returns `500`, not a crash)." Supersedes the original "crash on bugs you own" thesis; crash-vs-survive now lives in [Fault tolerance](fault-tolerance.md).
[3] Postel's Law (the Robustness Principle): "be conservative in what you send, be liberal in what you accept." Cited for its *boundary* form — absorbing deviations in external data.
[4] Soft assertions — a testing idiom (e.g. TestNG) in which failures are recorded and the run continues; the source of the "record, don't abort" technique adapted here as "absorb, but log."
