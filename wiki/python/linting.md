---
type: python
title: Linting and formatting in Python
description: "Use ruff for both Python linting (ruff check) and formatting (ruff format); defaults are the floor, run locally and in CI. Ruff is not a type checker — that stays mypy's job."
tags: [python, linting, formatting, ruff, static-analysis, ci]
timestamp: 2026-07-09T17:05:00Z
---

Every Python codebase in this wiki's scope uses **ruff** for **both** linting
and formatting, run **locally and in CI**. ruff (Astral, implemented in Rust) is
fast enough to run on every save without thought, and it consolidates a whole
stack of legacy tools behind two commands and one config.

> **Why this is Python-only.** The convention is about *the tool choice* — ruff
> is Python-specific. The general idea of "run a linter and formatter" applies
> to every language, but each has its own tool; this page records which one this
> wiki standardises on for Python.

# Two roles: linting vs formatting

ruff ships two capabilities, invoked as separate commands. They are distinct
jobs and worth not conflating:

| Command | Role | What it does | Nature |
| ------- | ---- | ------------ | ------ |
| `ruff check` | **Linter** | Analyses code for *problems* — unused imports, undefined names, mutable default args, bug-prone patterns, style violations. Diagnoses violations; `--fix` auto-applies safe ones. | **Diagnostic / rule-based.** Tells you what's wrong; some fixes need a human decision. |
| `ruff format` | **Formatter** | Mechanically rewrites code to a canonical *layout* — indentation, line wrapping, quote style, spacing, trailing commas, blank lines. | **Deterministic / opinionated.** One canonical output; you accept the whole file, no line-by-line review. |

The short version: **the linter tells you what's wrong; the formatter makes it
look uniform.** They overlap only at the edges (both can sort imports); the
linter concerns correctness/conventions, the formatter concerns pure
presentation.

# What ruff replaces

Historically a Python project needed a small constellation of tools for this:

| Concern | Legacy tool(s) | ruff |
| ------- | -------------- | ---- |
| Linting (bugs, style, conventions) | `flake8` + dozens of flake8 plugins | `ruff check` |
| Formatting (canonical layout) | `black` | `ruff format` |
| Import sorting | `isort` | `ruff check --select I` (lint rule) / formatter integration |

ruff folds all of these into **one install, one config file, one Rust binary**.
That removes a class of friction: no version drift between tools, no
flake8/black/isort disagreeing about line length, and no multi-second lint runs.

# Ruff is not a type checker

ruff does **linting and formatting only**. It is **not a type checker** —
verifying type correctness is [mypy's](type-safety.md) job, and ruff does not
duplicate it. The two are **complementary and non-overlapping**:

| Tool | Concerns | Catches |
| ---- | -------- | ------- |
| **mypy** (strict) | Types — shape, argument/return types, `Any` containment, narrowing. | A call passing a `str` where an `int` is required; an untyped boundary leaking `Any`. |
| **ruff** | Style, conventions, catchable bugs, layout. | An unused import; a mutable default argument; a line that isn't formatted to canonical form. |

Neither tool subsumes the other, so **both run in every project**: mypy proves
the type system holds (see [Type safety in Python](type-safety.md)), ruff keeps
the style and the bug-pattern surface clean. One subtlety worth knowing: ruff's
linter and formatter are **explicitly designed not to fight each other** — which
is why some pycodestyle *style* rules (e.g. whitespace groups `E1`/`E2`/`E3`)
are intentionally **not** in the default lint set, since the formatter owns
layout. You run both; they don't contradict.

# Minimum pyproject.toml setup

The required configuration is a `[tool.ruff]` section. ruff's **defaults are the
floor** — same "floor, not a ceiling" stance as the [mypy setup](type-safety.md):
every project runs on at least the defaults, and projects are free to layer more
on top.

```toml
# pyproject.toml — minimum ruff setup. Defaults are the floor; extend as the
# project needs (selecting extra rule sets, per-file ignores, etc.).
[tool.ruff]
target-version = "py311"   # set to the project's minimum supported Python
line-length = 88           # default; shared by the linter and the formatter

# Linter — `ruff check`. The default rule set (Pyflakes `F` plus the pycodestyle
# error groups `E4`/`E7`/`E9`) is the floor. Opt into more rules via `select`:
# [tool.ruff.lint]
# select = ["F", "E4", "E7", "E9", "I", "UP", "B", "SIM"]   # uncomment to extend

# Formatter — `ruff format`. Black-compatible defaults; no config needed.
# [tool.ruff.format]
```

Two points on the minimum:

- **`target-version` is worth setting explicitly.** ruff will infer it from
  `project.requires-python` if present, but setting it explicitly (e.g. `py311`)
  is safer — it governs lint rules like the `UP` (pyupgrade) modernisation set
  and formatter behaviour, so an accidental mis-inference would silently weaken
  both. Set it to the project's *minimum* supported Python.
- **`line-length = 88` is the default**, shown for visibility. It is shared by
  the linter and the formatter so they agree on wrapping; don't set different
  lengths for the two or they will disagree.

The commented `[tool.ruff.lint]` and `[tool.ruff.format]` blocks show where
extension lives; the defaults alone satisfy the floor.

# Local and CI

ruff runs the same two commands everywhere; only the flags differ between local
and CI:

```sh
# Lint — diagnose violations.
ruff check            # local: add --fix to autofix the safe ones
ruff check            # CI:    fails on any violation (run without --fix)

# Format — canonical layout.
ruff format           # local: rewrites files in place
ruff format --check   # CI:    fails if any file isn't already formatted
```

The pattern mirrors the wiki's other automated checks (mypy runs locally **and**
in CI): locally you let the tools *fix* (`ruff check --fix`, `ruff format`); in
CI you let them *fail* (`ruff check`, `ruff format --check`) so an unlinted or
unformatted change can't land. **Both commands gate CI** — a formatting failure
is just as blocking as a lint failure.

# Rationale

- **One tool, not a stack.** Consolidating flake8 + black + isort (+ plugins)
  into ruff removes version drift, inter-tool disagreement, and config sprawl —
  one binary, one config, one thing to install and run.
- **Fast enough to run always.** Because ruff is Rust-fast, there's no reason to
  gate it behind "only on CI" — it runs on save, on commit, and in CI with no
  perceptible cost. That is what makes "defaults as the floor" actually hold.
- **Complementary with mypy, not redundant.** Linting/formatting and type
  checking catch *different* classes of problem; running both is the point, not
  overlap. Each removes a category of failure that the other can't see.
- **Formatter in CI is non-negotiable.** Letting formatting drift produces
  meaningless diffs and review noise; `ruff format --check` in CI keeps the tree
  canonical so every diff is about substance.

# See also

- [Type safety in Python](type-safety.md) — mypy does the type checking; ruff does the linting + formatting. Distinct, complementary — together they are the Python automated-checks stack.
- [Logging in Python](logging.md) — sibling Python convention (logging setup), same language-specific layer of the wiki.

# Citations

[1] Ingested convention directive (2026-07-09): "When coding in python I like to use `ruff` to manage linting in the project." Clarified on ingest to: use ruff for **both** linting (`ruff check`) and formatting (`ruff format`); ruff defaults are the floor; run locally **and** in CI. Personal engineering convention; no external URL.
[2] ruff documentation — ruff is a fast Python linter and formatter implemented in Rust by Astral: `ruff check` (a flake8 replacement) and `ruff format` (a black-compatible replacement), configured under `[tool.ruff]` / `[tool.ruff.lint]` / `[tool.ruff.format]` in `pyproject.toml`. https://docs.astral.sh/ruff/
