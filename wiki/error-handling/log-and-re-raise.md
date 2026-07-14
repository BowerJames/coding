---
type: antipattern
title: "Log and re-raise — catching, logging, then re-raising an error"
description: "Catching an error, logging it, and re-raising it: double-logs (your line plus the handler's), drops the stack trace, and clutters intermediate frames. Log where handled — exactly once, at the handler; propagate silently between frames."
tags: [error-handling, logging, observability, antipattern, error-propagation]
timestamp: 2026-07-14T00:00:00Z
---

The smell: catch an error, log it, **then re-raise it** (or `throw` / `return Err`
again). You logged at an intermediate frame that the error was only going to pass
through on its way to a real handler.

```python
try:
    result = do_thing()
except SomeError as e:
    logger.error("do_thing failed: %s", e)   # logged here…
    raise                                     # …and re-raised
```

This is the negative form of the [Log vs. raise](log-vs-raise.md) rule — *"log
where handled, exactly once, at the handler."* Here you are not handling anything;
you are catching only to pass the error on, and adding a log line on the way. The
taxonomy calls that propagation (T3/T4 in [Error taxonomy](error-taxonomy.md)),
and propagation should be silent between frames.

> **Not the same as swallowing.** The opposite failure — catching and *not*
> logging at all — is the *silent-failure* anti-pattern (an error absorbed with no
> trail). Log-and-re-raise errs in the other direction: it logs too much, in the
> wrong place. The healthy middle is a single `ERROR`, at the handler.

# Why it's bad

Three harms compound:

**1. Double-logging.** If every frame on the bubble path logs-and-re-raises, one
error produces N log records — plus the boundary handler logs it once more when it
finally arrives. Deduplication breaks, alert counts inflate, and a single failure
can look like a cascade. The authoritative record is the handler's; every extra
line is noise on top of it.

```python
def layer_a():
    try:
        layer_b()
    except Error:
        log.error("layer_a caught it")   # record #1
        raise

def layer_b():
    try:
        layer_c()
    except Error:
        log.error("layer_b caught it")   # record #2
        raise

# …the boundary handler emits record #3 (with the trace). One failure, three lines.
```

**2. Trace loss / context poverty.** The hand-rolled line almost always drops the
stack trace. `logger.error(str(e))` (or `logger.error(e)`) prints the exception's
`str`, not where it came from — and the stack trace is *"the most important
debugging tool you will need later when figuring out what happened"* (Grinberg;
see [Citations](#citations)). The intermediate frame also rarely has the context
to write a *good* message: it caught to pass on, not to explain.

**3. Framework duplication.** When a framework already supplies the handler,
log-and-re-raise re-implements it per endpoint. The classic case is a Flask route
that wraps every database write in `try/except/rollback/logger.error/return 500`:

```python
# NOTE: this is how NOT to do it.
@app.route('/songs/<id>', methods=['PUT'])
def update_song(id):
    try:
        db.session.add(song)
        db.session.commit()
    except SQLAlchemyError:
        current_app.logger.error('failed to update song %s', song.name)  # traceless
        db.session.rollback()
        return 'Internal Server Error', 500
    return '', 204
```

Flask already catches the error, logs it **with a stack trace**, returns `500`,
and (via Flask-SQLAlchemy) rolls back the session. The route's block duplicates
all of that, worse, and is repeated in every endpoint that writes to the database.

# The fix

**Log where handled — exactly once, at the handler. Propagate silently between
frames.** If you are going to re-raise, do not log; let the error flow to the one
frame that actually handles it (recovers, or turns it into a clean boundary
outcome like `HTTP 500`), and let *that* frame emit the single `ERROR` with the
trace.

The Flask route above is a T4 ("do nothing") endpoint — the framework is the
handler:

```python
@app.route('/songs/<id>', methods=['PUT'])
def update_song(id):
    db.session.add(song)
    db.session.commit()      # let Flask catch, log (with trace), 500, and rollback
    return '', 204
```

No log line here, no `try`/`except`; Flask's `full_dispatch_request` is the
handler that logs once. (See [Error handling in Python](/python/error-handling.md)
and [Fault tolerance](fault-tolerance.md).)

# When it's acceptable

A *deliberate, single* breadcrumb at a specific architectural layer is the one
carve-out — e.g. marking "entered the billing boundary" or "request X failed
before it reached the handler". Two rules keep it honest:

- **Log it at `DEBUG`, not `ERROR`.** `ERROR` is the alertable channel
  ([Log output streams](/logging/streams.md)); emitting it here would duplicate
  the handler's authoritative `ERROR`. A `DEBUG` breadcrumb is non-alertable and
  survives dedup because it is a different severity.
- **Never as the default.** It is a conscious, documented choice at a named
  boundary, not something every `except` block does. If you find yourself
  log-and-re-raising routinely, you are not propagating — you are double-logging.

# In Python

- **`logger.exception()` belongs at the handler, not at the re-raise site.** It
  logs the message *and* the current exception's stack trace, so calling it at an
  intermediate frame still double-logs (just with a trace this time). Its right
  home is the boundary handler. See [Logging in Python](/python/logging.md).
- **`logger.error(str(e))` / `logger.error(e)` is the loaded form** of this
  anti-pattern in Python: it both double-logs *and* drops the trace. If a frame
  genuinely must log an exception, `logger.exception()` is the minimum — but
  prefer not logging at all and letting the handler do it.

# See also

- [Log vs. raise](log-vs-raise.md) — the rule this anti-pattern violates: log where handled, exactly once, at the handler.
- [Error taxonomy](error-taxonomy.md) — log-and-re-raise is mis-applied propagation (T3/T4); only the handler (recovery or boundary) logs.
- [Fault tolerance](fault-tolerance.md) — the single boundary catch-all that is the handler a log-and-re-raiser is usually duplicating.
- [Error handling in Python](/python/error-handling.md) — the Flask route before/after and `logger.exception()`.
- [Logging in Python](/python/logging.md) — `logger.exception()` vs `logger.error()` (the trace-loss mechanism).
- [Log output streams](/logging/streams.md) — why a breadcrumb belongs at `DEBUG`, and the handler's `ERROR` is the single alertable record.

# Citations

[1] Miguel Grinberg, "The Ultimate Guide to Error Handling in Python" (2024-10-07), https://blog.miguelgrinberg.com/post/the-ultimate-guide-to-error-handling-in-python — source of the trace-loss point (*"this particular log lacks information, especially the stack trace… use `logger.exception()` instead of `logger.error()`"*), the bad-Flask-endpoint / framework-duplication example, and the "let errors bubble up / handle at the top" principle. **Note:** the article does *not* name "log-and-re-raise" and does not discuss double-logging at intermediate frames — those are a wiki generalisation (see [2]).
[2] Wiki synthesis / user directive (2026-07-14): "add an antipattern for logging and re-raising errors." The double-logging-across-frames framing and the promotion of the smell to a named anti-pattern consolidate what was already noted inline in [Log vs. raise](log-vs-raise.md), [Error taxonomy](error-taxonomy.md), and [Error handling in Python](/python/error-handling.md).
