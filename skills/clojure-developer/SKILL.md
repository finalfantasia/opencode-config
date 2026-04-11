---
name: clojure-developer
description: |
  Use when the task is general Clojure software development rather than a
  narrow subtask: implementing features, debugging behavior, refactoring code,
  reviewing Clojure changes, or guiding REPL-driven development. This skill
  coordinates the Clojure workflow and directs you to clojure-eval,
  clojure-repl, and clojure-delimiter-repair as needed.
---

# Clojure Developer

This skill is the top-level workflow for Clojure development tasks.

For coding work, follow this sequence:

1. Explore and understand existing code
2. Use [clojure-eval](../clojure-eval/SKILL.md) first to prototype behavior and verify assumptions in a running REPL.
3. Use [clojure-repl](../clojure-repl/SKILL.md) for interactive exploration patterns such as `doc`, `source`, `dir`, edge-case checks, and iterative debugging.
4. Edit files only after behavior and assumptions have been prototyped and verified in the REPL.
5. If an edit introduces delimiter errors, run [clojure-delimiter-repair](../clojure-delimiter-repair/SKILL.md) immediately before making further edits.
6. Reload the namespace and verify the final behavior in the REPL after each meaningful change.

## Core Rules

- The REPL is mandatory for coding tasks. Prefer `REPL -> test -> edit -> reload -> verify`.
- Validate happy path, edge cases, and failure cases before considering a change complete.
- Favor small, composable functions and data-oriented transformations.
- Do not hide errors with silent fallbacks unless the user explicitly wants that behavior.
- Never treat untested code as done.

## When To Reach For Other Skills

- Use [clojure-eval](../clojure-eval/SKILL.md) to connect to or evaluate code in nREPL.
- Use [clojure-repl](../clojure-repl/SKILL.md) when you need exploration and debugging techniques.
- Use [clojure-delimiter-repair](../clojure-delimiter-repair/SKILL.md) when delimiter errors appear.

## Response Style

- Be concise and direct.
- Prefer concrete code or REPL output over long explanation.
- For "how should I approach this?" questions, explain the approach first, then demonstrate it in the REPL when useful.
- For concrete implementation or debugging requests, move directly into REPL-driven verification and execution.
