---
type: pattern
title: "EAFP vs LBYL — check before, or act and handle? (it depends on the language)"
description: "Two styles for operations that can fail: Look Before You Leap (check preconditions first) and Easier to Ask Forgiveness than Permission (act, then handle the failure). Exception-based languages favour EAFP; value-returning languages (Rust, Go) make explicit checking idiomatic. Let errors flow where the language allows."
tags: [error-handling, eafp, lbyl, toctou]
timestamp: 2026-07-10T16:20:00Z
---

When an operation can fail, there are two places to put the handling: *before*
you act (check the preconditions) or *after* (attempt the act and deal with the
failure). The two idioms are traditionally named by their unpronounceable
acronyms **LBYL** and **EAFP**. Which one is "right" depends on the language;
this page gives the cross-language rule.

# LBYL — Look Before You Leap

Check that the conditions for success hold, then act:

```
if can_do_x():
    do_x()
else:
    handle_error()
```

Deleting a file, LBYL-style:

```python
if os.path.exists(file_path):   # check…
    os.remove(file_path)        # …then act
else:
    print(f"Error: file {file_path} does not exist!")
```

LBYL has two structural weaknesses:

1. **You must enumerate every failure mode in advance.** A missing file is one
   reason a deletion fails, but so are: it's a directory; you don't own it;
   it's read-only; the volume is mounted read-only; another process locked it.
   You cannot realistically pre-check all of them, so LBYL tends to handle only
   the obvious case and miss the rest.
2. **Time-of-check to time-of-use (TOCTOU) race.** Between the check and the
   act the world can change, so a passing check no longer guarantees success —
   a classic source of subtle concurrency bugs.

# EAFP — Easier to Ask Forgiveness than Permission

Attempt the act, then handle whatever failure the callee reports:

```
try:
    do_x()
except SomeError:
    handle_error()
```

Deleting a file, EAFP-style:

```python
try:
    os.remove(file_path)        # act…
except OSError as error:        # …then handle
    print(f"Error deleting file: {error}")
```

The callee is responsible for detecting and reporting failure, so the caller no
longer has to enumerate failure modes, and there is no check/act race window —
the failure is reported at the moment it actually occurs.

# Which wins depends on the language

"EAFP is preferable" is **a statement about exception-based languages**, not a
universal law. Map it to your language's culture:

| Language family | Idiomatic style | "Let it bubble" looks like |
| --- | --- | --- |
| **Exception-based** (Python, Java, JS/TS, C#) | **EAFP.** Failures travel as exceptions; catching them and letting the rest bubble is the natural T4 from the [taxonomy](error-taxonomy.md). | (do nothing) |
| **Value-returning** (Rust `Result`/`?`, Go `if err != nil`) | **Explicit checking.** Errors are *values* you branch on; the LBYL-looking check `if err != nil` *is* idiomatic and is **not** an antipattern. | `?` (Rust) / `return err` (Go) |

The genuinely cross-language kernel is narrower than "EAFP wins":

> Where the language supports **transparent bubbling** — exceptions that flow up
> the stack, or `?`/`return err` that propagates a value with one token — prefer
> **letting errors flow** over hand-checking every precondition. You can't
> enumerate every failure mode, and the TOCTOU window only exists when you
> separate the check from the act. In a value-returning language the "check" is
> *receiving* the error value from the callee, which is not a TOCTOU risk, so
> the explicit style is fine.

So: in Python, reach for EAFP. In Rust/Go, the explicit `match`/`?`/`return err`
style *is* your EAFP — it's how errors flow.

# When you handle, catch the narrowest failure set

Whatever the style, when you *do* recover (a T2 in the
[taxonomy](error-taxonomy.md)), handle the **specific** failure the callee
raises, not "whatever happens":

- Catch the concrete error class / variant — `except OSError`, not a blanket
  handler. Errors you didn't list will then bubble honestly to a layer that can
  deal with them.
- Reserve a **catch-all** for exactly one place: the single boundary handler
  that guarantees no failure crashes the process (see
  [Fault tolerance](fault-tolerance.md)). A catch-all anywhere else silences
  real bugs — most defects surface as unexpected exceptions, and a mid-stack
  catch-all hides them.

# See also

- [Error taxonomy](error-taxonomy.md) — the 2×2 that decides *what* to do with an error (this page decides the *catching style*).
- [Error handling in Python](/python/error-handling.md) — EAFP in Python, including `except` specifics and why bare `except:` is forbidden.
- [Fault tolerance](fault-tolerance.md) — the one boundary handler where a catch-all is correct.
- [Log vs. raise](log-vs-raise.md) — log where handled, never swallow silently.

# Citations

[1] Miguel Grinberg, "The Ultimate Guide to Error Handling in Python" (2024-10-07), https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python — source of the LBYL/EAFP contrast and the "catch the narrowest set" rule (presented there for Python).
