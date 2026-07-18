---
name: init-harness-teams
description: "Use this skill when starting a brand-new coding project and the user wants Claude Code to scaffold a structured harness built on real parallel subagents instead of one Lead Agent switching roles — Planner/Builder/Reviewer as isolated .claude/agents/*.md subagents (build-state tracked by a deterministic script, not a fourth subagent), each pinned to a model tier chosen to conserve subscription quota, plus the same WAT-based file structure, test-driven self-improvement loop, and mandatory intake with a conditional clarify gate. Trigger on the same phrases as init-harness ('start a new project') when the user explicitly asks for subagent orchestration or wants roles split across models to save quota. Do not trigger for one-off scripts or existing-code fixes."
argument-hint: "[optional: one-line description of what you want to build]"
---

# Project Harness Initialization — Subagent Variant

Lead Agent orchestrating Planner/Builder/Reviewer as isolated subagents, each pinned to a model tier; build-state tracking (`PROGRESS.md`) is a deterministic script, not a fourth subagent. Interview the user before writing a file. If `$ARGUMENTS` was given, treat it as an answer to Question 1 and confirm it rather than re-asking.

## GROUND RULES (permanent for this project)

1. **Spanish to the user, English in every artifact.** Code, `.md` files, commits, subagent definitions — always English; never Spanish in an artifact, never English to the user.
2. **Ask, don't assume.** One question at a time, wait for the answer.
3. **Unsure user → propose 2-3 approaches with trade-offs**, wait for their choice — never pick architecture for them.
4. **No file before plan approval.** Intake → plan → approval → scaffold.
5. **Every file readable in ~2 minutes.** One responsibility per file, split if it grows past that; `CLAUDE.md` loads every session, keep it leanest of all.
6. **Never over-provision a model.** Ask "does this task need this much reasoning?", not "would more help." Escalate only on demonstrated failure, except Planner's one-time Opus default (see MODEL ASSIGNMENT).

## PHASE 0 — INTAKE (one question at a time, in Spanish)

1. **Objective** — what it does, for whom.
2. **Type** — one-off script, recurring automation, or tool with a UI.
3. **Stack** — propose 2 options with trade-offs if unsure.
4. **I/O** — expected inputs and outputs.
5. **Success criterion** — concrete, testable.
6. **Constraints** — paid APIs, credentials, limits; ask subscription vs. API billing here — decides whether tiering saves quota or dollars (see MODEL ASSIGNMENT).

**Clarify gate**: trigger if Q2 is "recurring automation"/"tool with UI", or any answer allows two contradictory implementations. Up to 5 follow-ups, one at a time, on the ambiguous areas only.

**Requirements checklist** (any unchecked item blocks Phase 1): Completeness — all 6 answered concretely · Clarity — no answer allows two implementations · Consistency — the stack (Q3) can produce the I/O (Q4) · Testability — the success criterion (Q5) is checkable.

## THE WAT PRINCIPLE

**Probabilistic AI handles reasoning; deterministic code handles execution.** Chained agent-improvised steps compound error — push execution into scripts so agents stay focused on orchestration.

- **Workflows** — SOPs in `workflows/`: objective, inputs, tools, outputs, edge cases.
- **Agents** — the subagent team below, each reading the relevant workflow first.
- **Tools** — Python scripts in `tools/`; check for an existing one before writing a new one. Secrets only in `.env`.

Applies even when a small project doesn't need all three folders — the separation is the point, not the folder count. Phase 1 decides how much to instantiate.

## PHASE 1 — PROPOSE THE STRUCTURE (after intake, before building)

- **Recurring automation / data pipeline** → full WAT (`workflows/`, `tools/`, `.claude/agents/`) + `PROGRESS.md`, `tools/checkpoint.py`, `.claude/settings.json`, `.claude/commands/checkpoint.md`.
- **One-off script or small tool** → skip `workflows/` and those five — no build state worth tracking. Still use Planner once (plan + self-audit) and Builder/Reviewer subagents per task.
- **Tool with UI** → full WAT + all five + whatever frontend/backend the UI needs.

```
CLAUDE.md                       # Trunk file, preloaded each session, lean.
memory.md                       # Self-healing log, written by Reviewer subagent.
PROGRESS.md                     # Build state tracker. Full-WAT/UI only.
init.py                         # Pre-flight check. Multiplatform.
.claude/agents/planner.md       # Subagent definition, model: opus.
.claude/agents/builder.md       # Subagent definition, model: sonnet.
.claude/agents/reviewer.md      # Subagent definition, model: sonnet.
tools/convert_to_markdown.py    # Optional — only if Q4 has non-Markdown inputs.
tools/get_context.py            # Deterministic memory/progress lookup by task-id. Full-WAT/UI only.
tools/update_progress.py        # Flips PROGRESS.md state; re-verifies before writing "done". Full-WAT/UI only.
tools/validate_state.py         # Format-consistency check, run by init.py pre-flight. Full-WAT/UI only.
tools/checkpoint.py             # Commits + pushes. Full-WAT/UI only.
archive.md                      # Archived memory.md entries, size-triggered. Full-WAT/UI only, on demand.
.claude/settings.json           # Stop hook wiring for checkpoint.py. Full-WAT/UI only.
.claude/commands/checkpoint.md  # Manual /checkpoint entry point. Full-WAT/UI only.
.gitignore                      # Created at the GitHub step.
```

Explain each file in one line, then wait for approval.

## THE MULTI-AGENT MODEL (parallel subagents, model-tiered)

Three subagents, each a standalone `.claude/agents/<name>.md` with its own frontmatter (`name`, `description`, `model`, `tools`) and system prompt. The Lead Agent never adopts their hats — it invokes via the Agent tool and relays results. New ones can be authored later on a catalog miss (see SKILLS & AGENTS CATALOG below).

Subagents share no conversation memory, so every invocation must be self-contained — but the Lead Agent must never hand-summarize `memory.md` into the prompt (that's a judgment call the WAT principle says belongs in a tool, not in reasoning). It passes only the task description and stable `task-id`; Builder and Reviewer's first action is running `python tools/get_context.py <task-id>` themselves and treating its output as complete starting context. Every subagent still verifies against the actual files on disk rather than trusting the handoff.

### Model assignment

| Subagent | Model  | Runs                | Why                                     |
| -------- | ------ | -------------------- | ---------------------------------------- |
| Planner  | opus   | once per plan         | One-time, highest-leverage call.        |
| Builder  | sonnet | once per task          | Correctness-critical, runs per task.    |
| Reviewer | sonnet | once per task          | Sole gate before memory.md/PROGRESS.md. |

**Retry ladder, same task** — no unbounded loop: fail once → Builder retries with the concrete failure in its prompt. Fail twice in a row → Lead Agent may ask the user for one-off permission to re-run that call on Opus (never bump a default tier without asking). Fail three times → **STOP**, no further automatic retries, ask the user for manual intervention.

Q6 API billing → note in the proposal that tiering also cuts dollar cost. Q6 subscription → note it stretches the shared usage window instead.

### Orchestration flow

1. **Planner** runs once: task list with success criteria, a stable `task-id`, a `scope: <name>` tag (e.g. `auth`, `pipeline`, `ui`), and `depends-on: <task-id>|none`. Lead Agent transcribes this into the initial `PROGRESS.md` (Planner is read-only, never writes files). Same scope vocabulary carries into that task's later memory.md entries. Logs a pass/fail self-audit naming anything cut (coverage, sequencing).
2. **Builder** runs once per task. After `get_context.py`, on full-WAT/UI runs `python tools/update_progress.py <task-id> in-progress`. Implements, existing tools first; **writes the test for the task's success criterion when it's testable** — Reviewer needs something concrete to check. `depends-on` tasks wait for their dependency's Reviewer pass; `none` tasks may fan out as **parallel Builder calls in the same turn**. **Gate**: ask the user before the first parallel fan-out — concurrent Sonnet calls draw down the shared usage window faster in a burst than the same tokens spent sequentially.
3. **Reviewer** always runs one call at a time, even when Builder fanned out — never parallelize it, concurrent writers to `memory.md` would race. Verifies each Builder output in turn, sole writer to `memory.md`.
4. Fail → correction loop (Retry ladder above): the concrete failure goes in the next Builder prompt, Builder fixes it (never re-running a paid call without asking first), Reviewer re-verifies.
5. Pass, full-WAT/UI → Reviewer's **last action**: `python tools/update_progress.py <task-id> done --test-cmd "<command>"` (or `--no-test` only if genuinely untestable). The script re-runs that command itself and only writes `[x]` on a real exit 0 — Reviewer's own claim is never enough by itself, the same rule Reviewer already applies to Builder. Lightweight (no `PROGRESS.md`) skips this, Lead Agent just advances.

## THE SELF-IMPROVEMENT LOOP (test-driven, not manual notes)

`memory.md` is fed by Reviewer, not filled in by hand. On failure: Reviewer identifies what broke from the actual files/tests (never trusting what it was told changed) → Builder fixes it → Reviewer re-verifies → lesson appended in this fixed format:

```
## [date] — <short title>
- Scope: <component/module tag — must match this task's scope in PROGRESS.md>
- What failed:
- Root cause:
- Fix:
- How to avoid it next time:
```

`Scope` is what makes this file machine-filterable — Reviewer reuses the task's exact `scope` from `PROGRESS.md`, by charter convention, so `get_context.py` can key an exact match instead of guessing from prose. If the failure traces to a `workflows/*.md` SOP, Builder fixes that too in the same pass (never overwrite a workflow without asking, otherwise).

**Planner reads `memory.md` in full**, once, at plan time — the one deliberate full read, justified by Ground Rule 6/Model assignment. **Builder and Reviewer never read it directly** on full-WAT/UI projects — they get their slice from `get_context.py <task-id>` (see CONTEXT RETRIEVAL below). On lightweight projects (no `get_context.py`), Builder and Reviewer read `memory.md` directly, in full, at the start of each task invocation instead — same file, same rule ("verify from the source"), just no deterministic tool standing in front of it at that scale. Not a one-time read at project start: every task invocation re-reads it, since a lesson from task 2's failure must be visible to task 5's Builder even without `get_context.py` to key it by scope. User corrections get appended here too, same format, by Reviewer.

## PROGRESS.md — BUILD STATUS (full-WAT / UI projects only)

Builder and Reviewer get this via `get_context.py <task-id>` rather than reading the whole file. Line content is written once, at scaffold time, by the Lead Agent transcribing Planner's list; after that only `update_progress.py` flips the state marker — Builder to `in-progress`, Reviewer to `done` — never a direct hand-edit, never Planner.

**States: `[ ]` pending · `[~]` in progress · `[x]` done and verified.** Fixed line format so `get_context.py` can parse it reliably:

```
[ ] T5 (scope: auth, depends-on: T3) — Add login endpoint
```

`depends-on: none` when there's no dependency. Answers "where did we leave off" — not a changelog, not a place for reasoning (that's `memory.md`).

## CONTEXT RETRIEVAL & STATE TOOLS

**`tools/get_context.py <task-id>`** (optional `--include-archive`, also searches `archive.md`) — deterministic replacement for hand-summarizing `memory.md`; keys off `task-id` instead of a line number that will have shifted by read time. Steps: (1) look up `task-id` in `PROGRESS.md` — not found → clear error, exit non-zero, a bad handoff `task-id` is a real bug and must surface loudly; (2) extract that task's `scope`, print the raw `PROGRESS.md` line; (3) grep `memory.md` for matching `Scope:` entries — no match → explicit "no matching history for scope `<name>`", never silent empty output; (4) `--include-archive` also searches `archive.md`. Run by Builder/Reviewer as their first action (both already have `Bash`, no permission change needed); Planner doesn't use it. Full-WAT/UI only.

**`tools/update_progress.py <task-id> in-progress`** — low-risk marker flip, informational, run by Builder right after `get_context.py`.

**`tools/update_progress.py <task-id> done (--test-cmd "<command>" | --no-test)`** — the state that matters, what the Lead Agent trusts to mean the build is green. Never takes Reviewer's word for it: one of the two flags is required, and the script **re-runs the command itself** before writing anything — only a real exit 0 produced by its own execution flips `[x]`; a failing or missing command leaves `PROGRESS.md` untouched and exits non-zero. `--no-test` is explicit, never a silent default. Mirrors the rule Reviewer already applies to Builder ("verify the actual files, never trust what you were told"), applied back onto Reviewer's own claim of success. Run by Reviewer as its last action, only after independently passing the task.

**`tools/validate_state.py`** — run by `init.py`'s pre-flight pass. Checks every `memory.md` entry has a `Scope:` line and every `PROGRESS.md` line matches the fixed format. A mismatch is a real bug (silently breaks `get_context.py`'s matching) — follows `init.py`'s **stop, don't continue, ask for help**, unlike the auto-archiving step below, which is a normal pass.

## init.py — PRE-FLIGHT CHECK

`CLAUDE.md` must instruct running `python init.py` before any change. Verifies the folder/file structure exists (including `.claude/agents/`), required `.md` files are present and non-empty, tests pass, and — full-WAT/UI only — runs `validate_state.py`.

If any of that fails: **stop, don't continue, ask for help.**

**Auto-archiving** (full-WAT/UI only, `memory.md` keeps growing otherwise): if it exceeds ~15 entries, move the oldest to `archive.md` (create if needed) — except any entry whose `Scope:` matches a task still `[ ]`/`[~]` in `PROGRESS.md`, which stays regardless of age. A normal deterministic step, not a failure — runs silently as part of pre-flight. `archive.md` is never read by Builder/Reviewer in the normal flow, only via `get_context.py --include-archive`.

## SUBAGENT DEFINITION TEMPLATE

```markdown
---
name: <planner|builder|reviewer>
description: <one line — when the Lead Agent should invoke this subagent>
model: <opus|sonnet>
tools: <minimum tool set this role actually needs>
---

<Role charter: for Builder/Reviewer, first action is get_context.py <task-id>
on full-WAT/UI, or reading memory.md directly, in full, at the start of every
task invocation, on lightweight (Planner reads memory.md in full instead,
once, on either footprint); which state script each role runs and when
(Builder: in-progress; Reviewer:
done, only after independently verifying the task); verify from disk, never
trust the handoff prompt or a tool's output; what it hands back to the Lead
Agent.>
```

- **planner** — read-only: `Read`, `Glob`, `Grep`. It plans, it doesn't touch files.
- **builder** — `Read`, `Edit`, `Write`, `Glob`, `Grep`, `Bash` (tests, `get_context.py`/`update_progress.py`).
- **reviewer** — `Read`, `Glob`, `Grep`, `Bash` (same), `Edit` (memory.md only, by convention — the tool grant can't enforce it, state the restriction in its charter).

## SKILLS & AGENTS CATALOG — ON DEMAND ONLY

Distinct from this project's own team above — pre-built external Skills/subagents, checked in order: (1) local catalog `D:\PROYECTOS CLAUDE\AI-Agency\CLAUDE-PLUGINS` (`skills/<name>/SKILL.md` dozens of entries, `agents/<name>.md` ~14 entries — match by frontmatter description, glob/grep names first); (2) connected Anthropic Skills/MCP servers.

**When**: right after intake, in Phase 1, if Q1-Q4 point to a specialized domain (name the match, don't install yet); also on-demand whenever Planner/Builder hits a gap. **Installing** (after approval, same gate as any file creation): copy only the matched item, project-scoped — never the whole catalog, never global. Name collision with the team's own three → rename the incoming file, don't overwrite a team subagent. Use only if it genuinely helps, never preload "just in case."

**Authoring a new subagent (catalog miss)**: same approval gate as any file creation (Ground Rule 4) — name the specific gap, wait for approval before writing. Minimum tool set, model tier by Ground Rule 6's test (default sonnet, never opus by default). Purely mechanical, zero-judgment work → write a `tools/*.py` script instead of a subagent, that's the cheaper tier. Lead Agent creates `.claude/agents/<name>.md`, adds it to `CLAUDE.md`'s subagent list, reuses it for later matching tasks instead of authoring a near-duplicate.

## NON-MARKDOWN INPUT HANDLING (MarkItDown)

Non-Markdown docs (PDF/DOCX/PPTX/XLSX) get converted to Markdown before being read — never raw bytes, never bespoke parsing code.

```
pip install 'markitdown[pdf,docx,pptx,xlsx]'   # scope to what Q4 actually needs
```

Wrap as `tools/convert_to_markdown.py`, never called inline:

```python
from markitdown import MarkItDown
from pathlib import Path
md = MarkItDown(enable_plugins=False)
result = md.convert_local(input_path)   # convert_local only — never a URL
Path(output_path).write_text(result.text_content, encoding="utf-8")
```

**Trigger**: any non-Markdown file, any time — declared at Q4 or dropped in later. Builder runs it, creating the tool first if it doesn't exist yet.

## AUTOMATED CHECKPOINTING (full-WAT / UI projects only)

`tools/checkpoint.py` checks for a real change — an uncommitted `git diff` against the last commit whose content includes a task newly marked `[x]` in `PROGRESS.md` — and, if so, commits — pushes only if a remote is configured (no remote isn't an error, just skips the push). Requires git initialized before the team starts task 1.

1. `Stop` hook, **merged into** `.claude/settings.json` (never overwrite an existing file, only add this entry):

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [{ "type": "command", "command": "python tools/checkpoint.py" }] }
    ]
  }
}
```

2. Manual `.claude/commands/checkpoint.md`:

```markdown
---
description: Force a checkpoint now, without waiting for the Stop hook.
---

Run `tools/checkpoint.py`; report what it committed, or that there was nothing to commit.
```

Fires after Reviewer's `update_progress.py <task-id> done` call closes a task — a deterministic trigger, never a mid-task judgment call.

## GITHUB — RIGHT AFTER SCAFFOLDING, BEFORE ANY TASK WORK

Initialize git before the team starts task 1 — git itself, not a remote (see AUTOMATED CHECKPOINTING), is what `checkpoint.py`'s `Stop` hook needs from the first completed task on; initializing git later makes early checkpoints fail with nothing to commit into. `.gitignore` excludes `.env`, credentials, `token.json`, `.tmp/`, and any sensitive `memory.md` content (ask if unsure). First commit in English; give the exact push commands if remote.

## CLAUDE.md REQUIREMENTS

Staying lean, referencing other files rather than inlining them:

- Instruct `python init.py` before any change; stop and ask if it fails.
- Instruct Builder/Reviewer to run `get_context.py <task-id>` as their first action instead of reading `memory.md`/`PROGRESS.md` directly; Planner reads `memory.md` in full once, at plan time. On lightweight projects (no `get_context.py`), state that Builder/Reviewer read `memory.md` directly, in full, at the start of every task invocation — not just once per project. Note auto-archiving past ~15 entries is automatic, not manual.
- Instruct Builder to run `update_progress.py <task-id> in-progress` on start, and Reviewer `update_progress.py <task-id> done --test-cmd "..."` (or `--no-test`) only as its last action — the script re-verifies before writing, it doesn't trust the call.
- Describe the team (Planner/Builder/Reviewer), model per role and why in one line each, the parallel-Builder gate, and the retry ladder's hard cap (3 fails on the same task → stop, ask the user).
- State the language rule and the Skills/Agents/MCP on-demand rule, with the `CLAUDE-PLUGINS` catalog path.
- If non-Markdown inputs are in play, restate the MarkItDown rule — must hold every session, not just at scaffold time.
- If this project has `workflows/`, state Builder updates a workflow when a failure traces to the SOP itself, never overwriting one without asking first.
- If `checkpoint.py` exists, note commits happen deterministically after Reviewer's `done` call closes a task — not a mid-task judgment call.
- State Ground Rule 6 explicitly, so a future session doesn't quietly bump a tier "to be safe."
- `CLAUDE.md` is edited in place after scaffolding (not appended like `memory.md`, not status-tracked like `PROGRESS.md`) whenever a task adds an agent file, a standing tool, a top-level folder, or a rule change. Builder edits it in the same task, targeted, not a rewrite — push detail into the referenced file if it would break the 2-minute limit. Reviewer fails the task if the reference wasn't added.

## START NOW

Begin Phase 0. Ask question 1 in Spanish. Don't create any file until the plan is approved.
