# Changelog

Design decisions for `UNIVERSAL_PROMPT.md`, in the same evidence-based format the harness
itself uses for `memory.md` — what changed, why, and what it replaced. Once the harness runs
against real projects, failures found in production get logged here too, not just design
decisions made in the abstract.

---

## Unreleased

Nothing yet.

---

## v1.3.1 — Doc-density and agent-readability pass (both skills)

### init-harness-teams/SKILL.md compressed 390 → 242 lines

- **Decision:** cut the full Python skeleton for `get_context.py` (duplicated the
  step-by-step spec already stated in prose right above it) and removed all 18 `---`
  section dividers (redundant with the `##` headers already providing visual separation).
- **Why:** the file's own Ground Rule 5 ("every file readable in ~2 minutes... split if it
  grows past that") wasn't being applied to itself. Verified against a full guardrail
  checklist before and after the cut — no functionality, threshold, or clarification was
  lost, only redundant formatting and duplicated content.

### Human-oriented narrative removed from both SKILL.md files

- **What failed:** a `SKILL.md` is read by Claude Code executing it, not by a human, but
  both files had accumulated retrospective and comparative asides — "Replaces the old
  Scribe subagent," decorative header subtitles like "(the backbone of this harness)," "the
  actual payoff of this variant," "(the trunk file you will write)." None of these changed
  what the executing agent does; the Scribe references in particular risked the agent
  looking for or expecting a component that no longer exists in the design.
- **Fix:** cut from both `init-harness/SKILL.md` and `init-harness-teams/SKILL.md`.
  Rationale that helps the agent generalize to cases the spec doesn't cover explicitly
  (e.g. the WAT principle's causal explanation, or why Reviewer never trusts its own
  "done" claim) was kept — the line drawn was "explains a decision the agent needs to
  make" versus "narrates the file's own design history for a reader who isn't the one
  executing it."

---

## v1.3.0 — init-harness-teams: deterministic context retrieval replaces hand-summarized memory, Scribe replaced by a verified script

### Subagents had no way to get task-relevant context without a full read or a hand-summarized prompt

- **What failed:** the original self-containment rule for isolated subagents ("pass the
  relevant slice of memory.md... in the prompt") required the Lead Agent to decide, by
  judgment, what counted as "relevant" — a probabilistic decision doing work the harness's
  own WAT principle says belongs in a deterministic tool, not in reasoning.
- **Fix:** `memory.md` entries gained a required `Scope:` field; `PROGRESS.md` lines gained
  a fixed format carrying `task-id`, `scope`, and `depends-on`. `tools/get_context.py
  <task-id>` resolves both from that one stable key — Builder and Reviewer run it
  themselves as their first action instead of trusting a Lead-Agent summary or a line
  number that will have shifted by read time. `init.py` gained size-triggered
  auto-archiving to `archive.md` (~15 entries, keeping any entry whose scope matches a
  still-active task regardless of age) so the file this tool reads never grows unbounded.

### Scribe subagent removed — a full subagent round trip to flip one character was the exact case the WAT principle exists to avoid

- **Decision:** `tools/update_progress.py` replaces the Scribe subagent. Builder marks
  `in-progress` on start; Reviewer marks `done` as its last action.
- **Why:** in the sub-agent variant (unlike `init-harness`'s sequential role-switching,
  which costs nothing extra since it's the same conversation), Scribe was a genuine network
  round trip plus context overhead spent on a zero-judgment marker flip.
- **Guardrail added on review:** an external review of the first draft of this fix flagged
  a blind-trust gap — `done` must never take Reviewer's word for a pass. The final version
  requires `--test-cmd "<command>"` or an explicit `--no-test`, and the script **re-runs
  the command itself** before writing anything; only a real exit 0 produced by its own
  execution flips `[x]`. Without this, a Reviewer mistake or a hallucinated pass could
  silently corrupt build state and let the Lead Agent advance past broken code.

### Retry loop had no hard stop

- **What failed:** the Builder/Reviewer correction loop could, in principle, run
  indefinitely if the user never responded to the Opus-escalation ask at 2 failures.
- **Fix:** three-step retry ladder — retry once with the concrete failure in the prompt,
  ask the user for one-off Opus permission at 2 fails in a row, **STOP and ask the user**
  at 3 fails in a row on the same task. No further automatic retries past that point.

### Builder's TDD instruction was missing from the sub-agent variant

- **What failed:** `init-harness` explicitly tells Builder to write the test for a task's
  success criterion when it's testable; `init-harness-teams` only implied it via a `Bash`
  tool grant, never stated it.
- **Fix:** ported the instruction over verbatim into the Orchestration flow.

### tools/validate_state.py added

- **Decision:** run by `init.py`'s pre-flight pass; checks every `memory.md` entry has a
  `Scope:` line and every `PROGRESS.md` line matches the fixed format — follows `init.py`'s
  stop-on-fail rule, unlike auto-archiving, which is a normal pass.
- **Why:** the new `Scope`/`task-id` format is enforced by charter convention, not by the
  tool grant. A silent typo in either file would break `get_context.py`'s matching without
  anyone noticing until a subagent silently got no history for a scope that actually had
  some.

---

## v1.2.2 — CONTEXT.md removed

### CONTEXT.md dropped entirely

- **Decision:** the file, the copy-on-scaffold mechanism, and the session read-order rule
  for it are all removed. Reverses v1.2.0.
- **Why:** on review, most of its content was already redundant with `CLAUDE.md` Ground
  Rules (the language rule was stated in both places). What wasn't redundant — feedback
  style — didn't justify a dedicated file, a copy mechanism, a new read-order rule, and a
  "never modify" constraint to maintain. The cost was structural, not the one line of
  content.
- **What was NOT done:** the feedback-style line was not folded into `CLAUDE.md` Ground
  Rules as a replacement. It was cut, full stop. If it turns out to matter in practice, that
  is a one-line addition to an already-existing file, not a reason to reintroduce a
  dedicated one.

---

## v1.2.1 — First bugs found from actually running the skill

### MarkItDown didn't trigger on a file dropped mid-session

- **What failed:** a `.docx` was added to a project and the agent was asked to analyze it
  directly — it read nothing, converted nothing, and reported no instruction told it how to
  handle non-Markdown files in that situation.
- **Root cause:** the trigger was scoped to "if Q4 declared this" and to the Builder role
  specifically. A file dropped in later and read directly, outside that path, wasn't
  covered. The instruction also lived only in `SKILL.md`, which loads once at
  `/init-harness` invocation — it doesn't persist into later sessions the way `CLAUDE.md`
  does.
- **Fix:** trigger broadened to "any non-Markdown file, at any point, regardless of how it
  was introduced." `CLAUDE.md REQUIREMENTS` now explicitly requires the generated
  `CLAUDE.md` to restate this rule, so it holds every session, not just at scaffold time.

### Reviewer→Scribe handoff had an unresolved branch

- **What failed:** "hands off to Scribe (if this project has one)" never said what happens
  when there isn't one.
- **Fix:** both branches stated explicitly — lightweight projects have Reviewer log its own
  pass note; there's nothing to hand off.

### CONTEXT.md had no source path

- **What failed:** the file said it gets "copied from the skill's template" without saying
  where that template lives, making the copy step unexecutable as written.
- **Fix:** literal path added (`~/.claude/skills/init-harness/CONTEXT.md`), stated in both
  the file listing and the CONTEXT.md section itself.

### PROGRESS.md / roles/scribe.md / tools/checkpoint.py conditional logic was scattered

- **What failed:** the Q2 dependency for these three files was implied in three different
  places with no single governing statement.
- **Fix:** one explicit rule — they're a package, tied to Q2, present or absent together.

### PreCompact hook removed

- **Decision:** dropped entirely, not relocated. It was bundled under "AUTOMATED
  CHECKPOINTING" despite having nothing to do with checkpointing — a mislabeling that
  caused it to be misread as one combined mechanism with the `Stop` hook.
- **Consequence, stated plainly:** the harness now has **no** context-fullness warning
  mechanism at all. `PreCompact` was the only native substitute for the originally requested
  "warn at 60%" feature, and it was an imprecise one. If this capability is wanted again, it
  needs its own section — not bundled with checkpointing — and a decision on whether the
  imprecise native version is acceptable or a custom token-monitoring script is worth
  building.

---

## v1.2.0 — Scribe, PROGRESS.md, CONTEXT.md, automated checkpointing

### Scribe separated from Reviewer, from day one, in full-WAT/UI projects

- **Decision:** `roles/scribe.md` is its own file whenever `PROGRESS.md` exists, not merged
  into Reviewer. Reverses the v1.0.0 default of "keep merged until the file exceeds ~2
  minutes to read."
- **Why the reversal:** the original recommendation assumed the split was speculative
  complexity with no evidence it was needed. That assumption was wrong — Prism, a real
  running project, had already independently arrived at this exact separation. That's
  evidence, not speculation. The corrected rule ties the split to a condition already in the
  harness (does `PROGRESS.md` exist?) rather than a manual line-count check the user would
  have to perform on every project — a cheaper and more scalable trigger than the one v1.0.0
  proposed.
- **Guardrail kept:** Scribe only exists where `PROGRESS.md` exists (full-WAT/UI). In
  lightweight one-off scripts, there is no build state to track, so there is no Scribe file —
  avoids creating an empty role with nothing to do.

### PROGRESS.md — conditional, not universal

- **Decision:** added to the file set, gated on the same Q2 condition as `workflows/`.
- **Why:** real, non-redundant value (build-state snapshot for session start) confirmed by
  Prism's actual use of it — but a 20-line script has no build state worth tracking, so it
  doesn't earn the file.

### CONTEXT.md — generalized, static, written once, never modified

- **Decision:** a single template (`init-harness/CONTEXT.md`) copied verbatim into every new
  project at scaffold time. Not generated from Phase 0 intake answers. Never rewritten or
  appended to afterward.
- **Why generalized, not personalized per project:** the content that's actually reusable
  (identity, technical background, feedback style, dev environment, git conventions) is
  stable across every project the user starts — it doesn't change per repo. The
  project-specific content in the original Prism version of this file (stakeholders, a
  specific business goal) was excluded on purpose: a file that is "static and never
  modified" cannot, by definition, also hold information that varies per project. That
  information is gathered instead through the normal Phase 0 intake, per project, where it
  belongs.
- **Guardrail:** if this file starts accumulating project-specific content or gets edited
  mid-project, that's a design failure — it has silently become a second `memory.md`.

### Automated checkpointing via a Stop hook, not a usage-percentage hook

- **Decision:** `tools/checkpoint.py` (commit + push) triggered automatically by a Claude
  Code `Stop` hook after every turn, plus a manual `/checkpoint` command calling the same
  script.
- **Why the original "90% of Pro plan usage" trigger was dropped:** verified against current
  Claude Code hooks documentation — no hook event exposes subscription usage percentage.
  Hooks fire on discrete lifecycle events (tool calls, turn end, compaction), not continuous
  account-level metrics. There was nothing to hook into for that trigger; building around an
  unverified mechanism would have shipped a feature that doesn't work.
- **Context-fill warning:** implemented via the native `PreCompact` hook instead of a
  configurable "60%" threshold, for the same reason — no hook exposes a context-percentage
  metric. A precise percentage is buildable only via a custom token-monitoring script, which
  was explicitly not added by default, consistent with the harness's own "don't add tooling
  it hasn't earned" principle.

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
