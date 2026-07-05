# Changelog

Design decisions for `UNIVERSAL_PROMPT.md`, in the same evidence-based format the harness
itself uses for `memory.md` — what changed, why, and what it replaced. Once the harness runs
against real projects, failures found in production get logged here too, not just design
decisions made in the abstract.

---

## v1.1.0 — Packaged as a Claude Code Skill

### Repackaged from a pasted prompt to an installed Skill
- **Decision:** The harness moved from "paste `UNIVERSAL_PROMPT.md` as your first message"
  to `init-harness/SKILL.md`, installed once at `~/.claude/skills/init-harness/` and invoked
  with `/init-harness`.
- **Why:** A pasted prompt is indistinguishable from prompt-writing. A Skill is Claude Code's
  actual native mechanism for reusable agent behavior — installed once, invoked by name,
  loaded on demand via progressive disclosure. Same content, correct delivery mechanism for
  a tool meant to be a personal default across every future project.
- **What changed, what didn't:** All prior decisions (WAT, sequential roles, Clarify Gate,
  Requirements Checklist, Planner audit, test-driven `memory.md` loop, on-demand MarkItDown)
  carried over unchanged. Only the packaging and invocation mechanism changed.

---

## v1.0.0 — Initial harness

### Sequential roles instead of parallel subagents
- **Decision:** One Lead Agent adopts roles (Planner/Builder/Reviewer) sequentially by
  reading `roles/*.md`, instead of spawning parallel subagents.
- **Why:** Calibrated to a fixed-quota subscription (Claude Code Pro). Parallel subagents
  burn quota fast for marginal gain on personal-scale projects.
- **Known limit:** This is not a universal truth — on a BYOK runtime with native subagent
  delegation, the cost calculus changes and parallel execution may be worth revisiting.

### `init.py` instead of `init.sh`
- **Decision:** Pre-flight check script is Python, not bash.
- **Why:** Target environment is Windows + portability to a second machine. Bash requires
  WSL/Git Bash on Windows; Python runs natively and is already a project dependency.

### Self-improvement loop is test-driven, not manual notes
- **Decision:** `memory.md` is written by the Reviewer after a verified failure/fix cycle,
  in a fixed format (what failed / root cause / fix / how to avoid it) — not a freeform
  notebook the user fills by hand.
- **Why:** A notebook only captures what the human remembers to write down. A loop fed by
  actual test failures captures what the system actually got wrong.

### Clarify Gate is conditional, not mandatory
- **Decision:** Extra clarification questions only trigger for recurring automations, tools
  with a UI, or genuinely ambiguous answers — not for simple one-off scripts.
- **Why:** Unconditional clarification adds friction with no payoff on trivial projects. The
  cost of ambiguity scales with project complexity, so the gate should too.

### Requirements Checklist requires evidence per item, not a narrated summary
- **Decision:** Phase 0 closes with a checked/flagged list (completeness, clarity,
  consistency, testability), each with stated evidence — not a prose recap.
- **Why:** A prose summary can look complete while hiding an unresolved contradiction. A
  checklist with required evidence forces the agent to justify each claim.

### Planner self-audits for scope before handing off to Builder
- **Decision:** The Planner checks its own task list for scope creep, requirement coverage,
  and dependency ordering before Builder starts — logged as a real pass/fail, not a bare
  "pass."
- **Why:** Unrequested scope expansion is cheap to catch by re-reading a plan and expensive
  to catch after code is already written.

### Non-Markdown inputs are converted on demand, not scanned automatically
- **Decision:** `tools/convert_to_markdown.py` (wrapping MarkItDown) runs only when a
  specific task needs to read a PDF/Office file — never a blanket scan on `init.py`.
- **Why:** An automatic scan would convert files nobody asked about, violating the harness's
  own "ask, don't assume" rule.

### Scope: Claude Code only, not cross-tool
- **Decision:** The trunk file stays `CLAUDE.md`, read directly by Claude Code. The harness
  does not adopt the `AGENTS.md` open standard used by Codex, Cursor, and OpenClaw/Hermes-style
  agents.
- **Why:** Deliberate scope choice, not an oversight — evaluated and declined in favor of
  keeping the harness's design surface small and specific to one tool.

---

## Unreleased

Nothing yet. The next entry in this file should come from a real project run, not another
design iteration in the abstract — the harness has not been validated end-to-end.
