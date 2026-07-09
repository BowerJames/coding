---
type: python
title: "Streaming seams in Python — asyncio queue-backed Stream / StreamWriter pair"
description: "Implement the stream-seam pattern in Python with asyncio: a single-pass async-iterator Stream and a push/end StreamWriter linked by a shared _Core queue, created together by create_stream(). Modern typing throughout (PEP 695 generics, Self, __slots__)."
tags: [python, streaming, concurrency, asyncio, typing]
timestamp: 2026-07-09T23:05:00Z
---

Python implements the [Stream seam](/patterns/streaming.md) pattern with
`asyncio.Queue` as the connecting channel. The producer and consumer share one
queue wrapped in a private `_Core`; a `Stream` is the consumer side (a
single-pass `AsyncIterator`) and a `StreamWriter` is the producer side
(`push`/`end`). A `create_stream()` factory constructs the pair.

# The implementation

```python
from __future__ import annotations

import asyncio
from typing import Self


class _Core[TEvent]:
    """Shared queue state linking a :class:`Stream` to its :class:`StreamWriter`.

    ``None`` is the termination sentinel pushed by :meth:`StreamWriter.end`;
    it is safe because events are never ``None``.
    """

    __slots__ = ("queue", "done")

    def __init__(self) -> None:
        self.queue: asyncio.Queue[TEvent | None] = asyncio.Queue()
        self.done: bool = False


class Stream[TEvent]:
    """Consumer side of a stream: a single-pass ``AsyncIterator`` of events.

    Iterate with ``async for event in stream:``. Iteration ends after the
    producer's :meth:`StreamWriter.end` (the terminal ``done``/``error`` event
    is yielded *before* iteration stops, so the final message is always
    reachable). Single-consumer; not safe to iterate concurrently.
    """

    __slots__ = ("_core",)

    def __init__(self, core: _Core[TEvent]) -> None:
        self._core = core

    def __aiter__(self) -> Self:
        return self

    async def __anext__(self) -> TEvent:
        item = await self._core.queue.get()
        if item is None:
            raise StopAsyncIteration
        return item


class StreamWriter[TEvent]:
    """Producer side of a stream.

    Push every event (including the terminal ``done``/``error``), then call
    :meth:`end`. Both methods are idempotent no-ops once :meth:`end` has run.
    """

    __slots__ = ("_core",)

    def __init__(self, core: _Core[TEvent]) -> None:
        self._core = core

    def push(self, event: TEvent) -> None:
        """Enqueue an event.

        No-op once :meth:`end` has run.
        """
        if self._core.done:
            return
        self._core.queue.put_nowait(event)

    def end(self) -> None:
        """Signal end-of-stream. Idempotent; pushes after this are no-ops."""
        if self._core.done:
            return
        self._core.done = True
        self._core.queue.put_nowait(None)


def create_stream[TEvent]() -> tuple[Stream[TEvent], StreamWriter[TEvent]]:
    """Create a linked consumer/producer pair sharing one queue.

    A provider's ``stream()``-style function keeps the :class:`StreamWriter`
    and returns the :class:`Stream` to its caller::

        consumer, writer = create_stream()
        asyncio.create_task(_run(writer, ...))
        return consumer
    """
    core = _Core[TEvent]()
    return Stream[TEvent](core), StreamWriter[TEvent](core)
```

# The provider idiom

A provider that wants to expose a `stream()`-style API keeps the writer, spawns a
background task to pump events, and returns only the consumer to its caller:

```python
async def stream() -> Stream[str]:
    consumer, writer = create_stream[str]()
    asyncio.create_task(_run(writer, ...))   # producer fills the stream
    return consumer                           # caller gets the read side only


async def _run(writer: StreamWriter[str], ...) -> None:
    try:
        for event in produce_events():
            writer.push(event)
    finally:
        writer.end()                          # always close, even on error
```

`_run` keeps the `StreamWriter`; the caller of `stream()` never sees it, so it
cannot push into its own input. The `finally` block is the expected close path —
`end()` is idempotent, so it is safe to call even if the producer failed mid-way.

# Typing notes (modern Python)

The snippet leans on modern type features throughout:

| Feature | Where | Notes |
| ------- | ----- | ----- |
| **PEP 695 generics** | `class _Core[TEvent]:`, `class Stream[TEvent]:`, `def create_stream[TEvent]()` | The new type-parameter syntax. **Sets the Python floor at 3.12+** — this is the binding constraint. |
| `typing.Self` | `def __aiter__(self) -> Self` | Returns the iterator as itself. Requires 3.11+, already subsumed by the 3.12 floor. |
| `X \| None` unions | `asyncio.Queue[TEvent \| None]` | The modern union syntax (3.10+). |
| `__slots__` | every class | No `__dict__` per instance — keeps the per-stream/per-writer memory minimal and prevents accidental attribute creation. |
| `from __future__ import annotations` | top of module | Postponed evaluation of annotations. Harmless alongside PEP 695; included for consistency. |

Every function and method is annotated (parameters *and* return types), and no
`Any` appears anywhere — this is in keeping with the wiki's
[Type safety in Python](type-safety.md) stance without needing to repeat it
here.

# The `None` sentinel

End-of-stream is a single `None` pushed by `end()`. This is safe **only** because
of the documented invariant that **events are never `None`** — the sentinel owns
that value exclusively. `__anext__` checks for it:

```python
item = await self._core.queue.get()
if item is None:
    raise StopAsyncIteration
return item
```

Python's type system cannot cheaply express "TEvent excluding `None`" (you'd
need to constrain the type variable to forbid `None`, which is heavier than the
convention is worth), so this is **type-safety-by-convention**: the invariant
lives in the `_Core` docstring and must be upheld by every producer. See the
pattern page for the language-agnostic discussion of [termination via
sentinel](/patterns/streaming.md#termination-via-sentinel) and how
`Option`/`Result`/enum variants express the same thing in other languages
without reserving a value.

# Notes

- **Idempotent `push` / `end`.** Both check `_core.done` first and no-op if the
  stream is already closed, so a producer can never crash by pushing after
  `end()` and double-`end` is harmless. The `done` flag is a producer-side guard
  — the consumer detects end-of-stream from the `None` sentinel, not from the
  flag.
- **`put_nowait`, not `await put`.** The producer enqueues synchronously; `push`
  is not `async`. The queue is unbounded, so `put_nowait` never blocks; this
  means **no backpressure** in the basic form (see the pattern's
  [Limitations](/patterns/streaming.md#limitations)) — choose a bounded queue if
  you need flow control.
- **Not concurrency-safe to iterate.** A `Stream` is single-consumer; two
  concurrent `async for` loops over the same instance will steal items from each
  other. Fan-out needs a demultiplexer layered on top.
- **The terminal event is reachable.** Because `end()` pushes `None` *after* any
  final event, and `__anext__` yields each real event before checking the
  sentinel, the last pushed event is always delivered before iteration stops.

# See also

- [Stream seam](/patterns/streaming.md) — the language-agnostic design this
  implements (problem, structure, when-to-use, properties, limitations).
- [Type safety in Python](type-safety.md) — sibling Python convention; this
  snippet annotates everything and uses no `Any`, consistent with mypy strict.
- [Linting and formatting in Python](linting.md) — sibling Python convention.
- [Logging in Python](logging.md) — sibling Python convention.

# Citations

[1] Ingested personal convention (2026-07-09): the user's `asyncio`-based stream
seam snippet — `_Core` / `Stream` / `StreamWriter` / `create_stream`. Implements
the [Stream seam](/patterns/streaming.md) pattern. Personal engineering pattern;
no external URL.
