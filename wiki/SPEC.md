# Wiki Spec

## Purpose

A personal knowledge base of **both general and language specific software-engineering best practices** —
the durable principles, patterns, and conventions I want to apply consistently
across all the code I write, regardless of language or project.

It is meant to be **LLM-readable**: when an agent (or I) needs to know "how do I
do this well?", the answer should already be filed here, cross-linked, and
cited — not re-derived from scratch every time.

## Scope

**In scope**
- Cross-cutting engineering practices (testing, version control, code review, naming, error handling, refactoring, debugging).
- Design & architecture patterns (when to apply them, when not to).
- Language-agnostic principles (coupling, cohesion, complexity, abstraction).
- Language-specific principles (python, typescript, rust)
- Anti-patterns and "smells" worth naming and avoiding.
- Workflow & process (issue tracking, branching strategy, CI/CD hygiene).

**Out of scope (for now)**
- Project-specific or company-specific details (those live in their own repos).
- Ephemeral "how I configured tool X today" notes, unless they generalise.
- Detailed tutorials that are better left to official docs (link to them, don't copy).

## Page types

Use these `type` values in frontmatter. Keep the set small and stable.

| `type`       | What it is                                                              |
| ------------ | ----------------------------------------------------------------------- |
| `practice`   | A best practice / principle you should follow.                          |
| `pattern`    | A reusable design or architectural pattern, with when-to-use guidance.  |
| `antipattern`| A named smell or trap to avoid, often with the better alternative.      |
| `python`     | Python-specific conventions and best practices.                         |
| `typescript` | TypeScript-specific conventions and best practices.                     |
| `rust`       | Rust-specific conventions and best practices.                           |

## Frontmatter

Every concept document has YAML frontmatter delimited by `---` with a **required** `type` field. `title`, `description`, `tags`, and `timestamp` are optional.

`timestamp` records the **last-modified** time of the page (ISO 8601, UTC — e.g. `2026-07-09T13:00:43Z`); bump it whenever you edit the page. `index.md` and `log.md` are reserved files with no frontmatter, and `SPEC.md` is a meta file that also carries none.
