---
type: pattern
title: "Error taxonomy — new/bubbled × recoverable/not: the 2×2 that decides how to handle an error"
description: "Classify any error by origin (you raised it vs it bubbled up from a callee) and by recoverability (you can fix it here vs you can't), giving four handling strategies. Most code should do nothing and let non-recoverable errors flow to a single handler."
tags: [error-handling, error-propagation, recoverability, fault-tolerance]
timestamp: 2026-07-14T00:00:00Z
---

The mechanics of error handling (`try/except`, `Result`/`?`, `if err != nil`) are
small and well known. The hard part is *what to do with a given error*. This page
gives a decision aid that works in any language: classify the error on two axes
and read off the strategy. It refines [Log vs. raise](log-vs-raise.md), whose
absorb/propagate split maps onto the rows below (absorb = T1+T2, propagate =
T3+T4).

# The two axes

- **Origin** — did *you* discover the problem (**new**), or did a function you
  called report it to you (**bubbled-up**)?
- **Recoverability** — can the code holding the error *fix it and continue*
  (**recoverable**), or is continuing impossible at this level
  (**non-recoverable**)?

Crossing them yields a 2×2 with one strategy per cell:

| | **Recoverable** | **Non-recoverable** |
| --- | --- | --- |
| **New** (you found it) | **T1** — fix state, continue (no raise). | **T3** — raise / propagate. |
| **Bubbled-up** (a callee raised) | **T2** — catch + recover + continue. | **T4** — do nothing; let it bubble. |

# The four strategies

- **T1 — new, recoverable.** You found an inconsistency and you can correct it
  yourself. Do so and continue — no error enters the system. *Example:* a song's
  `year` is missing; set `year = 'Unknown'` and proceed.
- **T2 — bubbled-up, recoverable.** A callee raised; you know how to repair it.
  Catch the specific error, recover, continue. *Example:* `get_artist()` raises
  `NotFound`; catch it and `add_artist()` instead.
- **T3 — new, non-recoverable.** You found a problem you cannot fix at this
  level. Raise (in the appropriate form) and let it bubble; it becomes a T2 at
  whatever higher layer *can* recover. *Example:* a song has no name; raise
  `ValidationError('the song must have a name')`.
- **T4 — bubbled-up, non-recoverable.** A callee raised and you cannot fix it.
  **Do nothing** — let it bubble to a handler that can. *Example:* `new_song()`
  just calls `get_song_from_user()` then `add_song_to_database()` with no
  `try`/`except` at all.

# Most code should be T4 (do nothing)

T4 is not laziness or "ignoring" errors — it is the deliberate default. A
function deep in the call stack rarely has the context to *present* a failure
(is this a console app, a GUI, a web server?) or to recover from every
dependency's failure. By doing nothing it lets the error reach the one layer
that does have that context: the [handler](#mapping-to-your-languages-error-channel).
Design applications so **as much code as possible is T4**; concentrate real
handling (T2) and the boundary catch-all at the edges. The result is clean,
maintainable code whose business logic is not cluttered with error plumbing.

This is the same insight behind [Fault tolerance](fault-tolerance.md): the
process survives by default because a single boundary handler turns an uncaught
error into a clean request failure (`HTTP 500`) rather than a crash.

# Mapping to your language's error channel

"Bubbled-up" means "arrived through whatever channel your language uses." The
strategies translate directly:

| Strategy | Python | Rust | Go | TypeScript/JS |
| --- | --- | --- | --- | --- |
| Recover (T1/T2) | `except E: fallback()` | `unwrap_or_else(\|e\| …)` / `match` | `if errors.Is(err, X) { fallback }` | `catch (e) { fallback }` |
| New non-recoverable (T3) | `raise DomainError(...)` | `return Err(...)` | `return fmt.Errorf(...)` | `throw new DomainError(...)` |
| Let bubble (T4) | (do nothing) | `?` | `return err` | (do nothing / `throw`) |
| Boundary handler | top-level `except Exception` | `main()` → log + exit | `log.Fatal` / middleware | framework / `uncaughtException` |

> **Which style to *catch* in is language-dependent** — see
> [EAFP vs LBYL](eafp-vs-lbyl.md). In exception-based languages you catch and
> let bubble; in value-returning languages (Rust, Go) you propagate with `?` /
> `return err` and recover with explicit `match`/checks. Both are idiomatic.

# Logging

Log **where the error is handled**, not where it is merely caught-and-passed-on
(see [Log vs. raise](log-vs-raise.md) for the authority):

- **T1/T2 (recovered)** — log `ERROR` at the recovery site. This frame is the
  *only* place that ever sees the error (it never reaches the boundary), so the
  log is mandatory — skipping it is a silent failure.
- **T3/T4 (propagated)** — do **not** log at every intermediate frame (logging only to pass it on is the [log-and-re-raise](log-and-re-raise.md) anti-pattern). The error
  is logged **once, at the handler** that finally absorbs it or turns it into a
  clean boundary failure.

The invariant "every failure is logged `ERROR`" (see
[Fault tolerance](fault-tolerance.md)) still holds — it means *exactly once, at
the handler*.

# See also

- [Log vs. raise](log-vs-raise.md) — absorb vs propagate and the logging discipline; this taxonomy refines it (absorb = T1+T2, propagate = T3+T4).
- [EAFP vs LBYL](eafp-vs-lbyl.md) — the *catching style* question; which is idiomatic depends on the language.
- [Fault tolerance](fault-tolerance.md) — survive by default; the boundary catch-all is what makes T4 safe.
- [Error handling in Python](/python/error-handling.md) — the Python realization of these four types.
- [Log and re-raise](log-and-re-raise.md) — the named anti-pattern for the T3/T4 logging mistake: catching, logging, then propagating anyway.

# Citations

[1] Miguel Grinberg, "The Ultimate Guide to Error Handling in Python" (2024-10-07), https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python — source of the new/bubbled × recoverable taxonomy and the four handling strategies.
