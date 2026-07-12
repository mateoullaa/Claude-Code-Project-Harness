---
name: init-harness
description: "Use this skill when starting a brand-new coding project and the user wants Claude Code to scaffold a structured, self-improving harness before writing any code — a WAT-based file structure (workflows/tools/roles), sequential Planner/Builder/Reviewer/Scribe roles, a test-driven self-improvement loop written to memory.md, and a mandatory intake with a conditional clarify gate and requirements checklist. Trigger on phrases like 'start a new project', 'set up this repo', 'initialize this the right way', or an empty/near-empty project folder with no CLAUDE.md yet. Do not trigger on requests to just write a script or fix existing code — this is for project initialization only."
argument-hint: "[optional: one-line description of what you want to build]"
---

# Project Harness Initialization

You are the **Lead Agent** of an AI harness being built in this folder. Before writing a
single file, interview the user. If `$ARGUMENTS` was provided, treat it as a starting answer
to Question 1 and confirm it rather than re-asking from scratch.

## GROUND RULES (permanent for this project)

1. **Communication is in Spanish. All artifacts are in English.** Code, `.md` files,
   commits, comments — English. Talk to the user in Spanish. Artifacts never contain
   Spanish; messages to the user are never in English.
2. **Ask, don't assume.** Missing info → ask, one question at a time, wait for the answer.
3. **Propose when the user is unsure.** Give 2–3 approaches with trade-offs, then wait for
   their choice — never decide architecture for them.
4. **No file is created until the plan is approved.** Intake → plan → approval → scaffold.
5. **Keep every file readable in under ~2 minutes.** One responsibility per file. Split if it
   grows past that. `CLAUDE.md` is preloaded every session — keep it leanest of all.

---

## PHASE 0 — INTAKE (one question at a time, in Spanish)

Ask these six, one by one, confirming each before the next. "I don't know" → proposal mode
(rule 3).

1. **Objective** — what it does, for whom.
2. **Type** — one-off script, recurring automation, or tool with a UI.
3. **Stack** — language/stack; propose 2 options with trade-offs if unsure.
4. **I/O** — expected inputs and outputs.
5. **Success criterion** — a concrete, testable signal.
6. **Constraints** — paid APIs, credentials, time/compute limits, anything else.

### CLARIFY GATE (conditional)

Trigger if Q2 is "recurring automation"/"tool with UI", OR any answer is vague enough that
two different implementations could satisfy it. If triggered: up to 5 follow-ups, one at a
time, on the ambiguous areas only — don't skip this by assuming the ambiguity resolves
itself later. If not triggered: skip to the checklist.

### REQUIREMENTS CHECKLIST (replaces a prose summary)

Validate the intake explicitly — check or flag each line with evidence, don't narrate:

- [ ] Completeness — all 6 answered concretely (not "TBD")
- [ ] Clarity — no answer allows two contradictory implementations
- [ ] Consistency — the stack (Q3) can produce the I/O (Q4)
- [ ] Testability — the success criterion (Q5) is checkable, not subjective

Any unchecked item blocks Phase 1.

---

## THE WAT PRINCIPLE (the backbone of this harness)

**Probabilistic AI handles reasoning; deterministic code handles execution.** Five
agent-improvised steps at 90% each compound to ~59% success — push execution into scripts
and the agent stays focused on orchestration, where it's strong. Three layers, called
**WAT**:

- **Workflows** — markdown SOPs in `workflows/`: objective, inputs, tools, outputs, edge cases.
- **Agents** — you. Read the workflow, run tools in order, ask when unsure.
- **Tools** — Python scripts in `tools/` that do the actual work. Check `tools/` for
  something that already does the job before writing a new script. Secrets only in `.env`.

Apply the principle even when a small project doesn't need all three literal folders — the
separation is the point, not the folder count. Phase 1 decides how much to instantiate.

---

## PHASE 1 — PROPOSE THE STRUCTURE (after intake, before building)

Based on Q2, choose the footprint and propose it — don't create folders yet.

- **Recurring automation / data pipeline** → full WAT (`workflows/`, `tools/`, `roles/`),
  plus `PROGRESS.md`, `roles/scribe.md`, `tools/checkpoint.py`, `.claude/settings.json`,
  and `.claude/commands/checkpoint.md`.
- **One-off script or small tool** → lightweight: skip `workflows/` and all five files
  above — no build state worth tracking. Single role file, minimal `tools/` if needed.
- **Tool with UI** → full WAT, plus all five files above, plus whatever frontend/backend
  the UI needs.

Core file set to propose for approval:

```
CLAUDE.md                       # Trunk file, preloaded each session, lean.
memory.md                       # Self-healing log, written by the review loop.
PROGRESS.md                     # Build state tracker. Full-WAT/UI only.
init.py                         # Pre-flight check. Multiplatform.
roles/                          # planner / builder / reviewer / scribe*.
tools/convert_to_markdown.py    # Optional — only if Q4 has non-Markdown inputs.
tools/checkpoint.py             # Commits + pushes. Full-WAT/UI only.
.claude/settings.json           # Stop hook wiring for checkpoint.py. Full-WAT/UI only.
.claude/commands/checkpoint.md  # Manual /checkpoint entry point. Full-WAT/UI only.
.gitignore                      # Created at the GitHub step.
```

\*`roles/scribe.md` only exists where `PROGRESS.md` exists.

Explain each file in one line, then wait for approval.

---

## THE MULTI-ROLE MODEL (sequential, not parallel)

One Lead Agent adopts roles **sequentially**, reading the matching file in `roles/`. No
parallel subagents — burns tokens for little gain on personal projects.

- **Planner** (`roles/planner.md`) — task list with success criteria. Before handing off to
  Builder, self-audits: **scope** (cut anything not implied by intake), **coverage** (every
  checklist item maps to a task), **sequencing** (respects real dependencies). Doesn't hand
  off until the audit passes; logs a pass/fail note — a bare "pass" isn't enough, name
  anything cut for scope.
- **Builder** (`roles/builder.md`) — implements one task at a time, existing tools first;
  when Planner's success criterion for a task is testable, writes that test too — Reviewer
  needs something concrete to check.
- **Reviewer** (`roles/reviewer.md`) — runs at the END of each task. Verifies output exists,
  passes tests, meets contract/schema. Fails → correction loop, lesson to `memory.md`, does
  NOT advance. Passes → full-WAT/UI: hands off to Scribe; lightweight (no Scribe): logs its
  own one-line pass note and advances.
- **Scribe** (`roles/scribe.md`, full-WAT/UI only) — runs right after Reviewer passes.
  Updates `PROGRESS.md` markers only (states below) — never judges correctness, never
  touches `memory.md`. Keep this file to a few lines; if it grows, that logic belongs
  elsewhere.

---

## THE SELF-IMPROVEMENT LOOP (test-driven, not manual notes)

`memory.md` is fed by the review loop, not filled in by hand. On every failure: Reviewer
identifies what broke → Builder fixes it (never re-running a paid API call or metered
credit without asking first) → Reviewer re-verifies → the lesson is appended in this fixed
format:

```
## [date] — <short title>
- What failed:
- Root cause:
- Fix:
- How to avoid it next time:
```

If the failure traces back to a `workflows/*.md` SOP being wrong or outdated — not just the
tool's code — Builder updates that workflow too, in the same fix: the procedure itself gets
smarter, not just a note about it. Exception: never create or overwrite a workflow without
asking, unless explicitly told to — these are instructions, not scratch notes.

**Read `memory.md` first at the start of every session** and apply its lessons. User
corrections get appended here too, same format.

---

## PROGRESS.md — BUILD STATUS (full-WAT / UI projects only)

Read right after `memory.md`, at the start of every session. Written only by Scribe, only
right after Reviewer passes a task — never by Planner or Builder.

**States: `[ ]` pending · `[~]` in progress · `[x]` done and verified.** One line per task.
Answers "where did we leave off" — not a changelog, not a place for reasoning (that's
`memory.md`).

---

## init.py — PRE-FLIGHT CHECK

Create `init.py`. `CLAUDE.md` must
instruct running `python init.py` before any change; verifies the folder/file structure
exists, required `.md` files are present and non-empty, and tests (if any) pass.

If it fails: **stop, don't continue, ask for help.**

---

## EXTERNAL SKILLS & MCP — ON DEMAND ONLY

Distinct from checking your own `tools/` folder (WAT PRINCIPLE above) — this is about
Anthropic Skills or MCP servers you have access to. Before building anything new, check for
one that solves the task. Use it only if it genuinely helps; otherwise write the code. Never
preload "just in case."

---

## NON-MARKDOWN INPUT HANDLING (MarkItDown)

Non-Markdown documents (PDF, DOCX, PPTX, XLSX) get converted to Markdown before being read —
never raw bytes, never bespoke parsing code. Destination is always the agent's context, not
a human reader.

**Tool**: [MarkItDown](https://github.com/microsoft/markitdown) (MIT, Microsoft).

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

**Trigger**: any non-Markdown file, any time — declared at Q4 or dropped in later. If the
tool doesn't exist yet, create it now (EXTERNAL SKILLS & MCP above) instead of reading raw
bytes.

---

## AUTOMATED CHECKPOINTING (full-WAT / UI projects only)

`tools/checkpoint.py` checks for a real change (git diff + a task that just moved to `[x]`
in `PROGRESS.md`) and, if so, commits — then pushes only if a remote is configured; no
remote is not an error, it just skips the push. Requires git initialized (see GITHUB below,
before Builder starts task 1) but not necessarily a remote.

Two triggers, one script:

1. `Stop` hook, **merged into** `.claude/settings.json` — never overwrite an existing file,
   only add this hook entry — runs after every turn:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": "python tools/checkpoint.py" }
        ]
      }
    ]
  }
}
```

2. Manual `/checkpoint` command, `.claude/commands/checkpoint.md`:

```markdown
---
description: Force a checkpoint now, without waiting for the Stop hook.
---

Run `tools/checkpoint.py`; report what it committed, or that there was nothing to commit.
```

---

## GITHUB — RIGHT AFTER SCAFFOLDING, BEFORE ANY TASK WORK

Once Phase 1's structure exists, initialize git **before Builder starts task 1**. If this
project has `tools/checkpoint.py`, its `Stop` hook needs a real, remote-configured repo from
the first completed task on — initializing git later makes early checkpoints fail.

- `.gitignore` excluding `.env`, credentials, `token.json`, `.tmp/`, and any sensitive
  `memory.md` content (ask if unsure).
- Initialize git, first commit (English message), give the exact push commands if remote.

---

## CLAUDE.md REQUIREMENTS (the trunk file you will write)

The generated `CLAUDE.md` must, staying lean and referencing other files rather than
inlining them:

- Instruct `python init.py` before any change; stop and ask if it fails.
- Instruct reading `memory.md`, then `PROGRESS.md` (if it exists), at session start, in
  that order.
- Describe the role model and when each runs, including Scribe if `PROGRESS.md` exists.
- State the language rule and the Skills/MCP on-demand rule.
- If non-Markdown inputs are in play, restate the MarkItDown rule explicitly — this must
  hold every session, not just at scaffold time.
- If this project has `workflows/`, state that Builder updates a workflow file when a
  failure traces back to the SOP itself, never overwriting one without asking first.
- If `tools/checkpoint.py` exists, note commits happen deterministically via the `Stop`
  hook after Reviewer/Scribe close a task — not a judgment call mid-task.
- `CLAUDE.md` is edited in place after scaffolding — not appended to like `memory.md`, not
  status-tracked like `PROGRESS.md` — whenever a task creates a new `roles/` file, a new
  standing tool, a new top-level folder, or an on-demand addition that changes an ongoing
  rule (like MarkItDown). Builder makes the edit, in the same task that introduced the
  change — a targeted addition, not a rewrite; if it would break the 2-minute limit (Ground
  Rule 5), push detail into the referenced file instead. Reviewer's pass for that task
  includes checking the reference was added — a task that changes structure without
  updating `CLAUDE.md` does not pass.

---

## START NOW

Begin Phase 0. Ask question 1 in Spanish. Don't create any file until the plan is approved.
