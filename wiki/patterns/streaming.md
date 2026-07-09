---
type: pattern
title: "Stream seam — split a stream into a consumer iterator and a producer writer sharing one queue"
description: "A provider returns an async stream it fills from a background task by splitting read/write authority at a seam: a single-pass consumer (async iterator) and a producer (writer) linked by one shared queue, created together by a factory."
tags: [streams, concurrency, async, producer-consumer, design-pattern]
timestamp: 2026-07-09T23:40:00Z
---

A common shape in async code: a provider must **return a stream to its caller**
while **internally producing the events** — often from a background task, a
callback, or an upstream async source. It cannot hand the caller the same handle
it writes to, or the caller could push into it (and the producer could be raced
by its own consumer). The read and write authority have to be **split at a
seam**, with the queue that connects them hidden behind two narrow interfaces.

This is the **stream seam** pattern. It is the async-stream analogue of keeping
a file's read and write ends as separate objects, and it is the clean way to
expose a `stream()`-style API whose body runs concurrently with the consumer.

> This is a design pattern — the durable, language-agnostic shape. The Python
> implementation lives in [Streaming seams in Python](/python/streaming.md).

# Structure

Four roles. Three are objects; the fourth is a factory that pairs them:

| Role | Responsibility |
| ---- | -------------- |
| **Core** (private) | Holds the **shared state** linking the two sides: the queue and a "done" flag. Never exposed to either side's caller — it exists only to couple one consumer to one producer. |
| **Consumer** (the read side) | A **single-pass `AsyncIterator`** over events. This is the object handed to the caller. Iterate with `async for`. |
| **Producer / Writer** (the write side) | Exposes **`push(event)`** and **`end()`**. Kept by the provider. |
| **Factory `create_stream()`** | Constructs one Core, wires both sides to it, and returns them as a **bound pair** (consumer + writer). The provider keeps the writer and returns the consumer. |

```
            create_stream()  ──►  consumer  +  writer     (a bound pair)
                                       │           │
                                       │           │ pushes / end()
                                       ▼           ▼
                                    ┌───────────────────┐
                                    │   Core (private)  │
                                    │   queue  +  done  │
                                    └───────────────────┘
```

The provider idiom is a one-liner per side:

```
# one create_stream() call yields the consumer and the writer together
spawn_background_task(writer, ...)   # producer fills the stream
return consumer                      # caller gets the read side only
```

The caller never sees the writer, and the writer never sees the iterator — the
seam enforces the division of authority.

# When to use

- **A function returns a stream it does not synchronously own.** You need to
  hand back an `AsyncIterator` immediately, then fill it from a background task,
  callback, or another async source (e.g. an MCP/LLP-style `stream()` that pumps
  tokens from a socket while the caller iterates).
- **You want to hide the queue.** Exposing a bare queue couples callers to its
  concrete type and lets them enqueue directly. The seam hides it behind a
  read-only iterator and a write-only writer.
- **Single consumer.** Exactly one caller iterates the stream to completion.

# Properties

The seam carries a small set of invariants that make it safe to use without the
producer and consumer coordinating:

- **Single-pass.** The consumer iterates the stream exactly once; it cannot be
  replayed.
- **Single-consumer.** Not safe for concurrent iteration. Fan-out (multiple
  consumers) needs a wrapper that demultiplexes — see [Limitations](#limitations).
- **Sentinel-terminated.** End-of-stream is signalled by a reserved sentinel
  value pushed by `end()` — see [Termination via sentinel](#termination-via-sentinel).
- **Idempotent close.** Once `end()` has run, both `push()` and `end()` become
  no-ops. A producer cannot crash by pushing after completion, and double-`end`
  is harmless.
- **The terminal event is always reachable.** Iteration stops *after* the
  end-of-stream sentinel is consumed, so any final event the producer pushed
  (including a logical "done"/"error" event) is yielded before iteration ends —
  the consumer never misses the last message.

# Termination via sentinel

End-of-stream is communicated by pushing a single reserved value (the sentinel)
onto the queue; the consumer treats receiving it as `StopAsyncIteration`. The
classic sentinel choice is **`None`**, made safe by the documented invariant
that **events themselves are never `None`** — the sentinel owns that value
exclusively, so it can never be confused with a real event.

Why a sentinel rather than a separate "closed" flag on the consumer? Because
the queue is the *only* channel between the producer and consumer, and the
consumer's only blocking primitive is "wait for the next item." A sentinel keeps
termination on that one channel — no extra signalling, no races between a flag
flip and a blocked `get()`. The `done` flag lives on the Core purely to make the
**producer** side idempotent (so `push`/`end` no-op after `end`); it is not what
the consumer reads to detect end-of-stream.

> **Other languages express this as a variant, not a stolen value.** Rust would
> model the queue as `TEvent` plus an `Option`/`Result`/dedicated `enum` end
> marker (no "reserved" value needed); TypeScript unions the element with a
> terminal discriminator. The sentinel technique is the dynamic-language form of
> the same idea — trading a cheap convention ("never `None`") for an exhaustive
> enum.

# Limitations

The basic seam is deliberately minimal, so it leaves a few concerns to the
caller:

- **No dedicated error channel.** `push` takes an event and `end` takes nothing,
  so a producer failure cannot be signalled as a distinct terminal condition in
  the bare form. Encode errors as an event variant (a logical "error" event
  pushed before `end`), or extend the sentinel to carry a `Result`-like value.
- **Backpressure depends on the underlying queue.** An unbounded queue offers no
  backpressure (the producer can run ahead without bound); a bounded one makes
  `push` potentially blocking and turns "producer runs ahead of consumer" into
  real flow control. Decide deliberately.
- **No fan-out.** Single-consumer by construction. Broadcasting to several
  consumers requires a demultiplexer that owns one writer and feeds N consumers
  — a separate pattern layered on top.
- **The producer and consumer lifetimes are coupled to the Core's.** There is no
  cancellation handshake here; cancelling the producer task and calling `end()`
  is the expected close path.

# See also

- [Streaming seams in Python](/python/streaming.md) — the `asyncio.Queue`-backed
  implementation of this pattern (`_Core` / `Stream` / `StreamWriter` /
  `StreamWiring` / `create_stream`), with modern typing (PEP 695 generics,
  `Self`, `__slots__`).

# Citations

[1] Ingested personal convention (2026-07-09): the user's `asyncio`-based stream
seam snippet — a `Stream` (consumer, single-pass `AsyncIterator`) and a
`StreamWriter` (producer, `push`/`end`) linked by a shared `_Core` queue,
created together by `create_stream()`. Personal engineering pattern; no external
URL.
