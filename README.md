# Claude Code Project Harness

Two Claude Code **Skills** that turn a blank folder into a structured, self-improving
development agent — with a real verification loop, not just a longer system prompt.

Install once. Run `/init-harness` (or `/init-harness-teams`) in any new project, forever.

## The problem this solves

Left alone, an agentic coding tool degrades in two predictable ways:

1. **Accuracy compounds downward.** If each unsupervised step is ~90% correct, five
   sequential steps land at ~59% success. Most "vibe coding" sessions fail here.
2. **Context saturates.** As a session grows, instructions buried early get diluted or
   forgotten — the agent gets _worse_, not better, the longer it runs.

This harness addresses both with one mechanism: **deterministic execution + a mandatory
review gate**, not more prompting.

## Why this is a Skill, not a prompt you paste

Claude Code Skills (`~/.claude/skills/<name>/SKILL.md`) are the current, native way to
package reusable agent behavior — installed once, invoked with `/name`, loaded on demand
instead of pasted every time. This repo ships the harness _as_ one, not as a block of text
to copy into a fresh chat:

- **Install scope matches the use case.** This is meant to be a personal default for every
  new project, so it's installed at the personal level (`~/.claude/skills/`), not
  re-copied per repo.
- **Progressive disclosure.** Claude Code reads the name and description at session start;
  the full body only loads when `/init-harness` actually runs — no wasted context on
  projects that don't need it.
- **One command, not one paste.** `/init-harness` in an empty folder. That's the interface.

## Two variants

- **`init-harness`** — one Lead Agent adopts Planner/Builder/Reviewer/Scribe roles
  sequentially, in the same conversation. No parallel subagents, no extra quota cost.
  Default choice for personal-scale projects.
- **`init-harness-teams`** — same WAT structure and self-improvement loop, but
  Planner/Builder/Reviewer are real, isolated `.claude/agents/*.md` subagents, model-tiered
  (Opus/Sonnet) to conserve subscription quota, with Builder able to fan out in parallel
  across independent tasks. Isolated subagents share no conversation memory, so build state
  (`PROGRESS.md`) and lesson history (`memory.md`) are retrieved deterministically by
  `tools/get_context.py`, keyed on a stable `task-id` — never a hand-summarized prompt.
  Reviewer's pass/fail state changes are independently re-verified by `tools/update_progress.py`
  before they're trusted, not taken on the subagent's word. Use this when you explicitly
  want subagent orchestration or model-tier separation, not for one-off scripts.

## How it works

- **Structured intake before any code is written** — six fixed questions, a conditional
  clarification gate for ambiguous or complex projects, and a requirements checklist that
  blocks progress until the intake is actually complete (not just answered).
- **WAT separation** (Workflows / Agents / Tools) — the agent orchestrates, deterministic
  Python scripts execute. This is _why_ accuracy doesn't compound downward: execution isn't
  left to probabilistic re-guessing on every step.
- **Sequential role model** (`init-harness`: Planner → Builder → Reviewer, plus Scribe in
  projects that track build state) instead of parallel subagents by default — same
  separation of concerns, without burning a fixed subscription quota. `init-harness-teams`
  trades that for real parallel subagents when quota isn't the binding constraint (see
  "Two variants" above).
- **A Reviewer that actually gates progress.** It runs at the end of every task, checks the
  output against a defined success criterion, and blocks advancement on failure. Every
  failure is logged in a fixed, greppable format in `memory.md` — read back by the agent at
  the start of every future session.
- **PROGRESS.md** — a build-state snapshot (`[ ]` / `[~]` / `[x]`), written by Scribe, read
  at the start of every session, so the agent knows where a project was left off without
  re-deriving it from the conversation history. Only provisioned for recurring automations
  and UI tools — a one-off script has no build state worth tracking.
- **On-demand tool use** — Skills, MCP servers, and document-to-Markdown conversion
  (via [MarkItDown](https://github.com/microsoft/markitdown)) are checked for and applied
  only when a task actually needs them, never preloaded.

## Installation

```bash
git clone https://github.com/mateoullaa/Claude-Code-Project-Harness.git
mkdir -p ~/.claude/skills
cp -r Claude-Code-Project-Harness/init-harness ~/.claude/skills/init-harness
# optional: the parallel-subagent variant
cp -r Claude-Code-Project-Harness/init-harness-teams ~/.claude/skills/init-harness-teams
```

That's it — `/init-harness` and/or `/init-harness-teams` are now available in every Claude
Code session, in any project, on this machine.

## Usage

1. Open a new, empty project folder and start Claude Code (`claude`).
2. Run `/init-harness` or `/init-harness-teams` (optionally with a one-line project
   description as an argument).
3. Answer the intake questions. No file is created until you approve the proposed structure.

## Repository structure

```
Claude-Code-Project-Harness/
├── README.md
├── CHANGELOG.md
├── init-harness/
│   └── SKILL.md           # sequential-role variant — copy this folder to ~/.claude/skills/
├── init-harness-teams/
│   └── SKILL.md           # parallel-subagent variant — copy this folder to ~/.claude/skills/
└── .gitignore
```

## Status

This is v1 of the harness. Both variants have been designed and internally reviewed across
several iterations, including external review (see `CHANGELOG.md`). Real testing has
started — the first pass on `init-harness` already surfaced and fixed a real bug (see
`v1.2.1`) — but **neither variant has yet been run end-to-end against a full production
project**. Partial validation, not full validation.

## Design log

Every non-trivial decision in this harness — why sequential roles by default and when the
subagent variant trades that for parallelism, why `init.py` over `init.sh`, why the review
loop writes to `memory.md` instead of the human, why this ships as a Skill instead of a
pasted prompt — is documented with its reasoning in `CHANGELOG.md`, not just its outcome.
