---
type: practice
title: "Task running with justfile — just is the single entry point; recipes wrap tool commands or native-language scripts"
description: "Every project standardizes on just (justfile) as the single uniform entry point for repetitive tasks, commands, and scripts. A recipe is situational — a bare tool command (e.g. usql, ruff check), an alias for a native-language script, or short shell. dotenv integration is always on; the same recipes run locally and in CI."
tags: [workflow, justfile, task-runner, automation, developer-experience, ci, dotenv]
timestamp: 2026-07-10T00:00:00Z
---

# The single entry point

Every project standardizes on **`just`** ([justfile](https://just.systems/)) as the
single entry point for repetitive tasks, commands, and scripts. Whatever the
language — Python, TypeScript, Rust — the way you *run* something is
`just <recipe>`, and the way you *discover* what you can run is `just` with no
arguments (which lists recipes). One tool, one place to look, one path from "I
just cloned this" to "I can run the thing."

This replaces the usual scatter: README snippets that drift, shell history that
lives in one developer's terminal, and project-specific `Makefile`s that conflate
a build system with a task launcher. `just` is deliberately only the latter — a
task runner — so it stays small even in projects that also have a real build
system.

# What a recipe can be (situational)

A frequent misunderstanding is that justfile is *only* a launcher for
native-language scripts. It isn't. A recipe is **whatever the task needs**, and
the right shape is situational:

| Recipe shape | When | Example |
| ------------ | ---- | ------- |
| **Bare tool command** | A developer tool already does the whole job. | `db: usql "$DATABASE_URL"` |
| **Alias for a native script** | The logic is non-trivial and belongs in the codebase's own language. | `seed: python scripts/seed.py` |
| **Short inline shell** | Genuinely one or two lines. | `serve: ./server --port 8080` |

The guiding rule: **when logic is non-trivial, move it into a native-language
script and have the recipe call it** — keeping `just` a *thin dispatcher* in that
case (a one-line `python scripts/foo.py`). But do not force a one-line tool
command through a script; if `usql` or `docker compose up` *is* the whole task,
the recipe is that command, directly.

Why the bias toward native scripts for real logic: justfile recipes are shell
strings, not a real programming language — they don't get the type checking,
testability, or tooling of the codebase's language. A long recipe turns the
`justfile` into an untested shell program written in a second language. A
one-line recipe that delegates to a tested script gets the best of both: `just`
as the uniform interface, the codebase's language and toolchain for the work.

# Always on: dotenv integration

dotenv loading is **always enabled**. Every justfile in scope begins with:

```just
set dotenv-load
```

so every recipe runs with the project's `.env` loaded into its environment. The
`.env` becomes the single source of truth for the environment configuration and
secrets the tasks need — a recipe like `db: usql "$DATABASE_URL"` simply reads a
variable the `.env` already provides, rather than each recipe re-specifying where
config comes from.

Related settings exist for the non-default cases: `dotenv-filename` (a custom
`.env` name, searched up the directory tree), `dotenv-path` (a fixed path that
errors if absent), `dotenv-override` (let `.env` win over an already-set
environment), and `dotenv-required` (error if no `.env` is found). The bare
`set dotenv-load` covers the common case.

# A minimum justfile

Illustrative, not prescriptive — only the `set dotenv-load` line is a firm rule:

```just
set dotenv-load

default:
    @just --list

# Checks — run locally and in CI.
lint:
    ruff check
typecheck:
    mypy

# Non-trivial logic lives in a native script.
seed:
    python scripts/seed.py

# A bare developer-tool command.
db:
    usql "$DATABASE_URL"
```

The shape of each recipe follows the situational table above: `lint`/`typecheck`
are bare tool commands; `seed` delegates to a native script; `db` is a one-line
tool invocation. Other conventions — a default recipe that lists tasks,
`@`-prefixed recipes to suppress echo, passing arguments — are project-dependent
and not prescribed here; reach for them when the project benefits.

# Local and CI

The same recipes run locally and in CI, so there is a single invocation path. CI
calls `just lint`, `just typecheck`, etc. — exactly what a developer runs —
rather than a parallel set of CI-only scripts that drift from local practice.
This mirrors the wiki's "run locally and in CI" stance for the automated checks
themselves: [ruff](/python/linting.md) and [mypy](/python/type-safety.md) are
invoked through these recipes in both places.

# Rationale

- **Discoverability.** `just --list` shows every available task; a new
  contributor needn't read the README or guess. One place to look.
- **Single source of truth.** "How do I run / seed / migrate / test this?" has
  one answer: the recipe. Not a README that drifted, not a teammate's shell
  history.
- **Reproducibility.** The recipe captures the exact invocation, including env
  via `.env`; "works on my machine" shrinks to "works in the recipe."
- **Language-agnostic.** The same tool and the same `just <recipe>` interface
  across Python, TypeScript, and Rust repos — the dispatcher is constant even
  when the native scripts differ.
- **CI parity.** One invocation path means local and CI can't drift apart; CI
  just calls the recipes.
- **dotenv centralises environment.** Secrets/config live in one `.env` consumed
  by every recipe, rather than being re-threaded into each.
- **Why `just` over `make`.** `just` is a modern, single-binary task runner (no
  build-system baggage) whose recipes aren't tab-sensitive and whose settings
  (dotenv, positional arguments, Windows shell) are first-class. `make` works,
  but it is a build tool pressed into task-running duty.

# See also

- [Linting and formatting in Python](/python/linting.md) — `ruff check` / `ruff format` are exactly the bare tool command a recipe wraps, run locally and in CI.
- [Type safety in Python](/python/type-safety.md) — `mypy` likewise; recipes are the uniform invocation layer for these checks.

# Citations

[1] Ingested convention directive (2026-07-10): "For calling repetitive tasks, commands and scripts I like to use justfile. The scripts may be written in the respective language of the codebase but that are normally aliased as just commands." Clarified on ingest to: `just` is the single uniform entry point for all repetitive tasks/commands/scripts; a recipe is situational — a bare tool command (e.g. `usql`, `ruff check`), an alias for a native-language script, or short inline shell, with non-trivial logic moved into a native script; **dotenv integration is always on** via `set dotenv-load`; the same recipes run locally and in CI. Personal engineering convention; no external URL.
[2] just documentation — `just` is a task runner that loads a `justfile` of recipes, each a shell command invoked as `just <recipe>`. dotenv loading is enabled with the `set dotenv-load` setting (default `false`); related settings are `dotenv-filename`, `dotenv-path`, `dotenv-override`, and `dotenv-required`. https://just.systems/man/en/ and https://github.com/casey/just
