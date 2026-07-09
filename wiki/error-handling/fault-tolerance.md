---
type: practice
title: "Fault tolerance — survive by default, fail narrowly, log every failure at ERROR"
description: "Applications should stay alive by default: fail at the narrowest scope that contains the damage (fallback → fail the request → crash only as a last resort). Every failure is logged at ERROR so developers are alerted, because it means an assumption of the application broke."
tags: [error-handling, fault-tolerance, reliability, observability, graceful-degradation]
timestamp: 2026-07-09T16:14:26Z
---

# The policy

Applications in this wiki's scope should **survive by default**. When something
fails, the goal is to keep the process alive and keep serving; a single failure
should not bring the whole application down. Two rules follow:

1. **Fail at the narrowest scope that contains the damage.** Recover with a
   *fallback* if you can; otherwise *fail the request* cleanly; only *crash the
   process* as a last resort (see [Blast radius](#blast-radius-choose-the-narrowest-scope)).
2. **Every failure is logged at `ERROR`.** A failure is always worth alerting
   on — even when you recovered from it — because it means an assumption of the
   application broke (see [What an `ERROR` means](#what-an-error-means)).

This is the *policy* layer. The per-case decision of whether to **absorb** an
anomaly (log + continue) or **propagate** it (throw / return `Err`) is a
separate, narrower question handled in [Log vs. raise](log-vs-raise.md); this
page decides *whether the process survives and at what scope you fail*.

# Blast radius: choose the narrowest scope

When a failure occurs, contain it at the smallest scope that keeps the rest of
the application correct:

| Scope of the damage | Action |
| ------------------- | ------ |
| **The request is satisfiable with a fallback.** | **Degrade gracefully**: log `ERROR`, serve the fallback (cached/stale data, a default, a reduced feature). Process alive, request succeeds. |
| **The request can't be satisfied, but process state is healthy.** | **Fail the request**: log `ERROR`, return a clean error to the caller (`HTTP 500`, `Result::Err`, an exception caught *at the request boundary*). Process stays up for the next request. |
| **Process-global state is corrupted and no safe fallback exists.** | **Crash / restart** (last resort): log `ERROR` first, then let the process exit so a supervisor / process manager restarts it. |

### The REST example

A REST server where something goes wrong internally while handling a request
should return **`HTTP 500`** for that request and keep serving — not crash the
whole server. The failure is contained at the *request* scope; the process
survives.

### When crashing is justified

Crashing the process is the **exception**, not the default. It is justified
only when *no safe fallback exists* **and** continuing would propagate corrupt
state to *other* requests (e.g. corrupted shared/global state that would make
every subsequent response wrong). Even then, **log `ERROR` before exiting** so
the failure stays observable and alertable. This is "fail fast" narrowed to its
last-resort role; the everyday stance is *fail softly*.

> Contrast with Erlang/OTP's *"let it crash"*: it crashes the *component*, and a
> supervisor restarts it so the *system* survives. The stance here generalises
> that to **request isolation** — a failing request is the component that
> "crashes" (into a `500`); the process is the system that keeps running.

# What an `ERROR` means

An `ERROR` means **an assumption the application relied on did not hold**. This
is a *diagnostic reading guide* for triaging an alert — when you are paged, ask
which assumption broke. Assumptions fall into three families:

| Family | The assumption that broke | Example |
| ------ | ------------------------- | ------- |
| **External-dependency** | A service/server exists, is reachable, and behaves per its contract. | A downstream API is down, times out, or returns an unexpected status/shape. |
| **Data-model** | Incoming or stored data conforms to your model (shape, type, presence, invariants). | A record has an unexpected field, or a type that drifted. |
| **Internal-state** | Your own objects/invariants hold (a state machine is in the expected state, a computed value is in range). | A transition is attempted from a state your code promised could not occur. |

> **A recovered failure is still an `ERROR`.** Serving a stale-cache fallback
> because the database died is an `ERROR`, not a `WARNING`: the request
> succeeded (fault tolerance doing its job), but an assumption broke and you want
> to know about it. Log it at error grade, then fix the producing system before
> the tolerated anomaly quietly becomes the new normal.

# `ERROR` vs `WARNING`

This refines [Log output streams](/logging/streams.md):

- **`ERROR`** — any failure / assumption-break, **including ones you recovered
  from** via a fallback. Alertable (stderr). Per the policy above, every failure
  is an `ERROR`.
- **`WARNING`** — *expected, in-course-of-business* soft handling where **no
  assumption actually broke**: a single scheduled retry, a brief rate-limit
  backoff. Not alertable (stdout).

The deciding question: **did an assumption break?** If yes → `ERROR`, even if
the caller got a good response via a fallback. If no (the code is doing
something it routinely anticipates) → `WARNING`.

# Related ideas

- **Graceful degradation** — the fallback tier of the blast-radius table;
  serving a reduced-but-useful result when the ideal path is unavailable.
- **"Let it crash" (Erlang/OTP)** — fail fast behind a supervisor; the component
  dies, the system survives. See the contrast above.
- **Soft assertions / log-and-continue** — recording a failure without throwing;
  see [Log vs. raise](log-vs-raise.md) for when to absorb vs propagate.
- **Circuit breakers / retries with backoff** — mechanisms that *implement*
  fallbacks and soft handling. Note: an *open* circuit (a dependency is failing)
  is itself an `ERROR`-grade signal, while a routine in-flight retry is
  `WARNING`.

# See also

- [Log vs. raise](log-vs-raise.md) — absorb recoverable anomalies vs propagate the rest (the mechanism; crash-vs-survive is decided here).
- [Log output streams](/logging/streams.md) — why `ERROR` is the alertable channel every failure belongs on.
- [Log line format](/logging/format.md) — the `<timestamp_utc> <level> <message>` shape error lines take.
- [Logging in Python](/python/logging.md) — language-specific implementation of the routing.
- [Type safety in Python](/python/type-safety.md) — mypy strict prevents the data-model / internal-state class of `ERROR` (broken shape/type assumptions) at analysis time, before they become runtime failures.

# Citations

[1] Ingested convention directive (2026-07-09): "Support fault-tolerant applications; all failures and unexpected behaviours should be logged as errors so developers can be alerted, since it means either the data model is incorrect or a third-party dependency is failing. Strive for the application not crashing; where possible reasonable fallbacks keep it alive (e.g. a REST server returns `500`, not a crash)." Personal engineering convention; no external URL.
[2] "Let it crash" — Erlang/OTP supervision philosophy: fail fast in the failing component, restart it behind a supervisor so the system survives. Contrasted here as request isolation.
[3] Graceful degradation — the general reliability principle of continuing in a reduced mode when the ideal path is unavailable; names the fallback tier of the blast-radius table.
