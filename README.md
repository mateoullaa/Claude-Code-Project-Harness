# Claude Code Project Harness

A single-file initialization prompt that turns Claude Code from a code editor into a
structured, self-improving development agent — with a real verification loop, not just
a longer system prompt.

## The problem this solves

Left alone, an agentic coding tool degrades in two predictable ways:

1. **Accuracy compounds downward.** If each unsupervised step is ~90% correct, five
   sequential steps land at ~59% success. Most "vibe coding" sessions fail here.
2. **Context saturates.** As a session grows, instructions buried early get diluted or
   forgotten — the agent gets *worse*, not better, the longer it runs.

This harness addresses both with one mechanism: **deterministic execution + a mandatory
review gate**, not more prompting.

## How it works

- **Structured intake before any code is written** — six fixed questions, a conditional
  clarification gate for ambiguous or complex projects, and a requirements checklist that
  blocks progress until the intake is actually complete (not just answered).
- **WAT separation** (Workflows / Agents / Tools) — the agent orchestrates, deterministic
  Python scripts execute. This is *why* accuracy doesn't compound downward: execution isn't
  left to probabilistic re-guessing on every step.
- **Sequential role model** (Planner → Builder → Reviewer) instead of parallel subagents —
  same separation of concerns, without burning a fixed subscription quota.
- **A Reviewer that actually gates progress.** It runs at the end of every task, checks the
  output against a defined success criterion, and blocks advancement on failure. Every
  failure is logged in a fixed, greppable format in `memory.md` — read back by the agent at
  the start of every future session.
- **On-demand tool use** — Skills, MCP servers, and document-to-Markdown conversion
  (via [MarkItDown](https://github.com/microsoft/markitdown)) are checked for and applied
  only when a task actually needs them, never preloaded.

## Usage

1. Open a new, empty folder in VS Code with the Claude Code extension installed.
2. Paste the entire contents of `UNIVERSAL_PROMPT.md` as your first message.
3. Answer the intake questions. No file is created until you approve the proposed structure.

## Status

This is v1 of the prompt. It has been designed and internally reviewed across several
iterations (see `CHANGELOG.md`) but **has not yet been run end-to-end against a production
project** — that validation is the next milestone, not a claim already made here.

## Design log

Every non-trivial decision in this prompt — why sequential roles over subagents, why
`init.py` over `init.sh`, why the review loop writes to `memory.md` instead of the human —
is documented with its reasoning in `CHANGELOG.md`, not just its outcome.
