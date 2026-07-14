---
type: python
title: "Error handling in Python"
description: "Python realization of the error taxonomy: try/except mechanics, EAFP, custom exception classes, logger.exception() for stack traces, no assert for error handling, one top-level except-Exception catch-all, and letting the framework (Flask/Tkinter) do it."
tags: [python, error-handling, exceptions, eafp]
timestamp: 2026-07-14T00:00:00Z
---

Python is an exception-based language, so [EAFP](/error-handling/eafp-vs-lbyl.md)
is the idiomatic style and the [error taxonomy](/error-handling/error-taxonomy.md)
maps directly onto `try`/`except`/`raise`. This page shows each of the four
taxonomy types in Python and collects the Python-specific rules that follow.
It summarises Miguel Grinberg's *"The Ultimate Guide to Error Handling in
Python"* (see [Citations](#citations)).

# The four types in Python

Using a running `add_song_to_database(song)` example:

**T1 — new, recoverable.** You found the problem; fix the state and continue. No
exception enters the system.

```python
def add_song_to_database(song):
    # the schema forbids NULL; default and carry on
    if song.year is None:
        song.year = 'Unknown'
    # ...
```

**T2 — bubbled-up, recoverable.** A callee raised; catch the *specific* error,
recover, continue. (This is [EAFP](/error-handling/eafp-vs-lbyl.md) in action.)

```python
def add_song_to_database(song):
    # ...
    try:
        artist = get_artist_from_database(song.artist)
    except NotFound:
        artist = add_artist_to_database(song.artist)  # recover
    # ...
```

**T3 — new, non-recoverable.** You cannot fix it here. Raise and let it bubble
until it reaches a layer that can (where it becomes a T2).

```python
class ValidationError(Exception):
    pass

def add_song_to_database(song):
    # ...
    if song.name is None:
        raise ValidationError('the song must have a name')
    # ...
```

**T4 — bubbled-up, non-recoverable.** A callee raised and you have no recovery.
**Do nothing** — let it bubble to the handler.

```python
def new_song():
    song = get_song_from_user()      # may raise (Ctrl-C, cancel, …)
    add_song_to_database(song)        # may raise (db offline, …)
```

`new_song()` doesn't know — and shouldn't — whether it runs under a console, a
GUI, or a web framework, so it has no business deciding how to *present* a
failure. That's a [separation-of-concerns](/error-handling/error-taxonomy.md#most-code-should-be-t4-do-nothing)
argument for leaving handling to the edges.

# Catch the narrowest exception

When you recover (T2), catch the concrete class(es), not everything:

```python
try:
    os.remove(file_path)
except OSError as error:     # good: specific
    ...
```

Two corollaries:

- **Never use a bare `except:`.** It catches *everything*, including
  `SystemExit`, `KeyboardInterrupt`, and bugs in your own code that ought to
  crash loudly. [PEP 760](https://peps.python.org/pep-0760/) removes bare
  `except:` from the language for this reason. If you mean "all ordinary
  exceptions", say `except Exception:`.
- **Reserve `except Exception` for one place only** — the single top-level
  boundary handler below. A catch-all anywhere up the stack silently swallows
  the unexpected exceptions that are almost always real bugs.

# Use custom exception classes

When no built-in fits, define a subclass so callers can catch *meaningfully*
(`class ValidationError(Exception)`) rather than string-matching on a message.
The choice of class is part of your function's contract — document what it can
raise, just as you document its parameters.

# Log with `logger.exception()`, not `logger.error(str(e))`

When you log at a handler, use [`logger.exception(msg)`](logging.md): it emits
the message **and the current exception's stack trace**, which is the single
most useful artifact for later debugging. `logger.error(str(e))` drops the
trace — and is exactly the context-poor pattern to avoid.

This is the Python mechanism for the "log where handled" rule: a recovered
(T1/T2) failure is logged at the recovery site; a propagated (T3/T4) failure is
logged **once, at the boundary handler**, with `logger.exception`.

# Don't use `assert` for error handling

`assert` is for documenting conditions you *expect* to be true (and for tests),
not for runtime error handling. It is **stripped entirely when Python runs with
`-O`**, so an `assert` in a production code path simply vanishes — your "check"
silently disappears in optimised builds, and a failed assert crashes the
process with no recovery. Raise a real exception (`raise ValueError(...)`,
`raise ValidationError(...)`) for any condition the program must actually
enforce.

# One top-level catch-all — and let the framework do it

Design the application so **no exception ever reaches the Python layer** (i.e.
nothing escapes as an uncaught traceback that crashes the process). Put a single
`except Exception` boundary at the top — the one place a catch-all is correct
(see [Fault tolerance](/error-handling/fault-tolerance.md)):

```python
import sys

def my_cli():
    ...

if __name__ == '__main__':
    try:
        my_cli()
    except Exception:
        logger.exception("unexpected error")   # log once, with trace
        sys.exit(1)
```

With this in place, the rest of the code can be T4 (do nothing) and let errors
flow — they are guaranteed to be caught, logged, and turned into a clean exit.

Application frameworks already provide this boundary, so you usually don't write
it yourself:

- **Flask** wraps every request in `full_dispatch_request()`, which catches
  exceptions, logs them with stack traces, and returns `HTTP 500` to the client.
- **Flask-SQLAlchemy** hooks into that handling to roll back the session on a
  database error.
- **Tkinter** wraps each event handler in a catch-all so a faulty handler can't
  crash the GUI.

So a Flask route that writes to the database can — and should — be pure T4:

```python
@app.route('/songs/<id>', methods=['PUT'])
def update_song(id):
    # ...
    db.session.add(song)
    db.session.commit()      # let Flask catch, log, 500, and rollback
    return '', 204
```

The per-endpoint `try`/`except`/`rollback`/`logger.error` block is an
anti-pattern here — the framework-duplication case of [log-and-re-raise](/error-handling/log-and-re-raise.md): it duplicates what the framework already does, and the
hand-rolled `logger.error('…', e)` invariably omits the stack trace.

# Dev vs prod from one boundary

Because handling is centralised at the boundary, switching behaviour between
environments is one line *there*, with no change to business logic:

```python
if __name__ == '__main__':
    try:
        my_cli()
    except Exception:
        if mode == "development":
            raise                       # dev: crash, see the full traceback
        logger.exception("unexpected error")
        sys.exit(1)
```

In development you *want* crashes and stack traces (so bugs get noticed and
fixed); in production the same boundary catches everything, logs it, and exits
cleanly. Frameworks expose this same switch as their debug/dev mode.

# See also

- [Error taxonomy](/error-handling/error-taxonomy.md) — the 2×2 these four Python types instantiate.
- [EAFP vs LBYL](/error-handling/eafp-vs-lbyl.md) — why EAFP is Python's idiomatic catching style.
- [Fault tolerance](/error-handling/fault-tolerance.md) — the policy the top-level catch-all implements (fail the request, not the process).
- [Log vs. raise](/error-handling/log-vs-raise.md) — log where handled, never swallow silently.
- [Log and re-raise](/error-handling/log-and-re-raise.md) — the named anti-pattern behind the per-endpoint try/except/logger.error block and the traceless `logger.error(str(e))`.
- [Logging in Python](logging.md) — `logger.exception()` and the stdout/stderr routing.

# Citations

[1] Miguel Grinberg, "The Ultimate Guide to Error Handling in Python" (2024-10-07), https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python — source for the taxonomy in Python, the narrowest-catch rule, custom exceptions, `logger.exception`, `assert` caution, the top-level catch-all, and the Flask/Tkinter examples. Comment thread additionally notes PEP 760 (no bare excepts) and the `logger.exception` recommendation.
