---
type: python
title: Type safety in Python
description: "Type safety is critical in Python: run mypy in strict mode, annotate every function, and avoid Any by receiving unknown data as object and narrowing it. Prefer dataclasses or pydantic as containers. Justify any type: ignore; prefer not to need one."
tags: [python, type-safety, mypy, static-analysis]
timestamp: 2026-07-09T17:05:00Z
---

Type safety is critical. Python is dynamically typed, so the interpreter will
*not* catch a type error until the line runs in production — the compiler-grade
guarantee has to come from somewhere else. In this wiki's Python code it comes
from **mypy in strict mode**: a static checker that proves type correctness
before the program ever runs.

> **Why this is Python-only.** Statically-typed languages (Rust, TypeScript)
> enforce type safety at the *compiler*, so they need no separate convention
> here — the language does it for free. Python lacks that, so the discipline
> has to be carried explicitly by a tool. There is intentionally no general
> `practice` page for "type safety"; this is the Python-specific implementation.

# The policy

Every Python codebase in this wiki's scope follows four rules:

| # | Rule |
| - | ---- |
| 1 | Run mypy in **strict mode** (locally **and** in CI). |
| 2 | **Annotate every function** — parameters *and* return type. |
| 3 | **Avoid `Any`.** Receive unknown data as `object` and **narrow** it. Prefer **dataclasses or pydantic** as containers. |
| 4 | `# type: ignore` only with an adjacent **justification**; prefer not to need one. |

Rule 3 is the one that does the most work and gets its own section:
[Avoiding Any — use object + type narrowing](#avoiding-any-use-object--type-narrowing).

# Minimum pyproject.toml setup

The required configuration is a single line under `[tool.mypy]`:

```toml
# pyproject.toml — minimum required setup. strict = true is the floor;
# add more (overrides, per-module flags, plugins) as the project needs.
[tool.mypy]
strict = true

# Recommended extras (beyond the minimum) — uncomment to harden further:
# disallow_any_explicit = true   # ban writing Any, forcing `object`
# warn_redundant_casts = true    # flag casts that change nothing
```

This is a **floor, not a ceiling**: `strict = true` is the minimum every project
must meet, and projects are free to layer more on top. `strict` is a bundle that
turns on a large set of flags; the ones that matter for the rules above:

- `disallow_untyped_defs` / `disallow_incomplete_defs` → enforce **rule 2**
  (every function fully annotated).
- `disallow_any_generics` / `warn_return_any` → enforce **rule 3** (`Any` does
  not silently propagate; bare generics like `list` must become `list[object]`).
- `warn_unused_ignores` → enforces **rule 4** (a `# type: ignore` mypy no longer
  needs is itself an error, so stale/unjustified ignores can't linger).

Two extras are recommended beyond the minimum (shown commented above):
`disallow_any_explicit = true` hard-bans writing `Any` in annotations, forcing
`object` for unknown values — it turns "avoid `Any`" from aspiration into
enforcement. `warn_redundant_casts = true` flags `cast()`s that don't actually
change the type (a sign the cast was unnecessary).

# Annotate every function

Every function takes typed arguments and returns a typed result. No exceptions
for "trivial" functions — strict mode (`disallow_untyped_defs`) treats a missing
annotation as an error.

```python
# Bad — untyped; strict mode rejects this.
def load(user_id):
    return fetch(user_id)

# Good — parameters and return type annotated.
def load(user_id: int) -> User:
    return fetch(user_id)
```

Local variables rarely need annotations (mypy infers them), but annotate
explicitly where inference would land on `Any` — see the boundary rule below.

# Avoiding Any — use object + type narrowing

`Any` is the type system's **opt-out**: a value of type `Any` is assignable
to and from anything with no checks, and it is **contagious**. If a function
returns `Any`, the variable receiving it is `Any`, and everything that touches
that variable becomes `Any` — type checking silently spreads its absence
throughout the call graph. `object` is the cure.

## `Any` vs `object`

| Type | Meaning | What you can do with it |
| ---- | ------- | ----------------------- |
| `Any` | "Trust me — could be anything." Checking is **off**. | Anything, unchecked. Contagious. |
| `object` | "Could be anything, but prove it first." Checking is **on**. | Only what `object` supports — you must **narrow** before use. |

`object` is the **top type**: every type is a subtype of `object`, so it
accepts any value, *but* it keeps you inside the type system. To call a method
or treat an `object` as a specific type you must narrow it first — which is
exactly the discipline that preserves the guarantee.

## The boundary: bind incoming `Any` to `object`

The external world is dynamically typed. `json.loads()` and
`httpx.Response.json()` both return `Any`, because a JSON payload's shape isn't
known statically. (The `httpx` library itself is fully typed — `Response`,
`Client`, etc. — it's only the *parsed payload* that is `Any`.) The rule:

> At the boundary, immediately bind incoming `Any` to an explicit **`object`**
> annotation to kill contagion, then narrow.

```python
import json

# Bad — payload is Any; the Any-ness now propagates to anything using payload.
payload = json.loads(raw)
print(payload["name"])          # unchecked; crashes at runtime if it's a list


# Good — payload is object; you must narrow before use, so checking stays on.
payload: object = json.loads(raw)
if isinstance(payload, dict):
    name = payload["name"]      # mypy still wants narrowing of the value…
```

The explicit `: object` is the load-bearing part. `payload = json.loads(raw)`
gives `payload` the inferred type `Any` (contagion); `payload: object = ...`
gives it the declared type `object` (contained). The same applies to
`response.json()`, external config, and any third-party call returning `Any`.

## The safety hierarchy: narrow, don't `cast`

Not every way of turning `object` into a concrete type gives the same guarantee.
There is a strict hierarchy:

| Technique | What it does | Guarantee |
| --------- | ------------ | --------- |
| `Any` | unchecked | **none** |
| `cast(T, x)` | *you* assert it's `T`; mypy trusts you | **asserted** (you own the risk) |
| `object` + `isinstance` / `TypeGuard` / `match` | proven at runtime **and** tracked statically | **verified** |

Only the last row actually guarantees type safety. `cast()` is an *unchecked
assertion* — mypy takes your word for it, so it provides no runtime guarantee
and should be a **last resort**, used only where genuine narrowing is impossible
(and justified like a `type: ignore`). The default is to **narrow with runtime
checks**, which the type checker tracks and the runtime confirms.

### Narrowing tools

- **`isinstance`** — narrows to a concrete type for simple cases.

  ```python
  payload: object = json.loads(raw)
  if isinstance(payload, int):
      reveal_type(payload)   # int
  ```

- **`TypeGuard` / `TypeIs`** (PEP 647 / PEP 742) — reusable narrowing
  predicates, the clean way to narrow into a `TypedDict` or custom shape.

  ```python
  from typing import TypeGuard, TypedDict


  class UserShape(TypedDict):
      id: int
      name: str


  def is_user_shape(v: object) -> TypeGuard[UserShape]:
      return (
          isinstance(v, dict)
          and isinstance(v.get("id"), int)
          and isinstance(v.get("name"), str)
      )


  payload: object = json.loads(raw)
  if is_user_shape(payload):
      reveal_type(payload)   # UserShape
  ```

  (`TypeIs`, available in Python 3.13+ / backported, narrows in *both* branches
  — the `else` branch knows it is *not* the target type.)

- **`match`** with class patterns (Python 3.10+) — structural narrowing.

  ```python
  payload: object = json.loads(raw)
  match payload:
      case {"id": int(id), "name": str(name)}:
          ...                 # narrowed
      case _:
          raise ValueError("unexpected payload shape")
  ```

## Worked example

```python
import json
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class User:
    id: int
    name: str


def parse_user(raw: str) -> User:
    """Parse a JSON string into a fully-typed User (no Any escapes)."""
    payload: object = json.loads(raw)        # kill Any contagion at the edge

    if not isinstance(payload, dict):
        raise ValueError("payload is not an object")

    user_id = payload.get("id")
    name = payload.get("name")
    if not (isinstance(user_id, int) and isinstance(name, str)):
        raise ValueError("payload has wrong shape")

    return User(id=user_id, name=name)       # concrete type flows outward
```

`parse_user` takes untyped input and returns a concrete `User`; callers never
see `Any`. Everything from this point on is statically checked.

# Data containers — dataclasses or pydantic

The preferred containers for typed data are **dataclasses** or **pydantic
models**. Narrow your boundary `object` into one of these.

| Container | When to use it |
| --------- | -------------- |
| **`@dataclass`** | Lightweight, stdlib, no dependencies. Use for **internal / already-typed** values where you want a typed, immutable (with `frozen=True`) container. It does **not** validate input — narrow *before* constructing it, as in the worked example above. |
| **pydantic model** | Use when a value **enters unvalidated from outside**. Pydantic validates at construction time, so it performs the `object`→typed narrowing *for you* in one step and hands back a typed instance. |
| `TypedDict` | For genuinely **dict-shaped** data you want to keep as a dict (e.g. passing through a structure without instantiating a class), narrowed via `TypeGuard`. |

The unifying principle: choose the container that matches where the data came
from — pydantic at unvalidated boundaries (it narrows for you), dataclasses for
the typed interior (lightweight), `TypedDict` for raw dict shapes.

# type: ignore discipline

A `# type: ignore` tells mypy to look the other way, so it must always carry a
**justification** — an adjacent comment naming *what* is being suppressed and
*why* it is safe:

```python
some_call()  # type: ignore[arg-type]  # upstream stub is wrong; tracked in #1234
```

`warn_unused_ignores` (part of `strict`) makes a stale ignore an error: once the
underlying issue is fixed, the ignore must go. The goal is to need **none** —
strict mode surfaces most real problems, and a forest of ignores usually means
stubs or abstractions that should be fixed instead.

# Rationale

Type errors are a class of runtime failure — shape, type, and model-variant
violations that are exactly the "broken assumption" an `ERROR` represents (see
[Fault tolerance](/error-handling/fault-tolerance.md)). Static checking removes
that entire class *before* runtime: the bug can't become an `ERROR` in
production because mypy refuses to let the code type-check. Strict mode matters
because it makes "annotate everything / no `Any`" **non-negotiable** rather than
aspirational — the guarantee holds only if there are no opt-outs.

# See also

- [Fault tolerance](/error-handling/fault-tolerance.md) — mypy strict *prevents* the data-model / internal-state class of `ERROR` (broken shape/type assumptions) at analysis time, before they become runtime failures.
- [Linting and formatting in Python](/python/linting.md) — ruff is the Python linter/formatter; it complements mypy (ruff is not a type checker, mypy is not a linter). The two run together as the Python automated-checks stack.
- [Logging in Python](/python/logging.md) — sibling Python convention (logging setup), same language-specific layer of the wiki.

# Citations

[1] Ingested convention directive (2026-07-09): "Type safety is critical in Python; the preferred type checker is mypy in strict mode. All functions should be annotated. `Any` should be avoided unless absolutely necessary. A `type: ignore` should be justified but it is preferred not to need one." Personal engineering convention; no external URL.
[2] Refinement (2026-07-09): avoid `Any` by receiving unknown data as `object` and **narrowing** it (`isinstance` / `TypeGuard` / `match`); prefer runtime narrowing (verified) over `cast` (asserted). Recommended config extras: `disallow_any_explicit`, `warn_redundant_casts`. Preferred data containers: dataclasses or pydantic.
