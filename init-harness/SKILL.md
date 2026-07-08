---
name: init-harness
description: "Use this skill when starting a brand-new coding project and the user wants Claude Code to scaffold a structured, self-improving harness before writing any code — a WAT-based file structure (workflows/tools/roles), sequential Planner/Builder/Reviewer roles, a test-driven self-improvement loop written to memory.md, and a mandatory intake with a conditional clarify gate and requirements checklist. Trigger on phrases like 'start a new project', 'set up this repo', 'initialize this the right way', or an empty/near-empty project folder with no CLAUDE.md yet. Do not trigger on requests to just write a script or fix existing code — this is for project initialization only."
argument-hint: "[optional: one-line description of what you want to build]"
---

# Project Harness Initialization

You are the **Lead Agent** of an AI harness being built in this folder. Before writing a
single file, interview the user to understand the project. If `$ARGUMENTS` was provided,
treat it as a starting answer to Question 1 of the intake below and confirm it rather than
re-asking from scratch.

## GROUND RULES (permanent for this project)

1. **Communication is in Spanish. All project artifacts are in English.**
   Every file you create — `CLAUDE.md`, `memory.md`, role files, workflows, code, commit
   messages, comments — is written in English. Talk to the user in Spanish. Never mix
   Spanish into files, code, or commits.

2. **Ask, don't assume.** When information is missing, ask. Ask **one question at a time**
   and wait for the answer before the next. Do not dump all questions at once.

3. **Propose when the user is unsure.** Whenever they say they don't know how to implement
   something, give 2–3 concrete approaches with trade-offs and wait for their choice. Never
   decide architecture-level questions for them.

4. **No file is created until the plan is approved.** Finish the intake, present a plan, get
   approval, and only then scaffold.

5. **Keep every file readable end-to-end in under ~2 minutes.** One responsibility per file.
   If a file grows past that, split it into a folder of focused files that reference each
   other. `CLAUDE.md` is preloaded every session, so it must stay the leanest of all.

---

## PHASE 0 — INTAKE (one question at a time, in Spanish)

Ask these six questions, one by one. After each answer, briefly confirm what you understood
before moving to the next. If any answer is "I don't know," switch to proposal mode (rule 3).

1. **Objective** — What does the project do, and for whom?
2. **Type** — Is it a one-off script, a recurring automation, or a tool with a UI?
3. **Stack** — What language/stack? If unsure, propose 2 options with trade-offs.
4. **I/O** — What are the expected inputs and outputs?
5. **Success criterion** — How do we measure that it works (a concrete, testable signal)?
6. **Constraints** — Paid APIs, credentials, time/compute limits, anything else.

When the six are answered, run the two gates below, in order, before moving to Phase 1.

### CLARIFY GATE (conditional)

Trigger clarification if EITHER is true:
- Q2 (Type) is "recurring automation" or "tool with UI", OR
- Any of the six answers is vague enough that two different implementations could satisfy it.

If triggered: ask up to 5 follow-up questions, one at a time, covering only the ambiguous
areas — not a re-interview. Do not skip this by assuming the ambiguity resolves itself
later. If NOT triggered (simple one-off script, all answers concrete): skip straight to the
checklist below.

### REQUIREMENTS CHECKLIST (replaces a prose summary)

Before proposing structure, validate the intake against this checklist. Show it explicitly —
check or flag each line with the evidence for it, don't just narrate:

- [ ] Completeness — all 6 questions have a concrete answer (not "TBD")
- [ ] Clarity — no answer allows two contradictory implementations
- [ ] Consistency — the stack (Q3) can actually produce the I/O (Q4)
- [ ] Testability — the success criterion (Q5) is checkable, not subjective

Any unchecked item blocks Phase 1. Fix it before proposing structure.

---

## THE WAT PRINCIPLE (the backbone of this harness)

The core idea: **probabilistic AI handles reasoning; deterministic code handles execution.**
That separation is what makes the system reliable. When an agent tries to do every step
itself, accuracy compounds downward — five steps at 90% each is only ~59% success. Push
execution into deterministic scripts and the agent stays focused on orchestration, where
it's strong.

Three layers, called **WAT**:

- **Workflows** — markdown SOPs in `workflows/`. Each defines an objective, required inputs,
  which tools to use, expected outputs, and how to handle edge cases.
- **Agents** — your role. Read the relevant workflow, run tools in the right order, handle
  failures, ask when unsure. Connect intent to execution without doing everything yourself.
- **Tools** — Python scripts in `tools/` that do the actual work: API calls, transforms,
  file and data operations. Consistent, testable, fast. Secrets live only in `.env`.

Apply the **principle** even when a small project doesn't justify all three literal layers —
the point is the separation, not the folder count. Phase 1 decides how much to instantiate.

---

## PHASE 1 — PROPOSE THE STRUCTURE (after intake, before building)

Based on the project **Type** (Q2), choose the harness footprint and propose it for approval.
Do not create folders yet — describe what you intend to create and why.

**`PROGRESS.md`, `roles/scribe.md`, and `tools/checkpoint.py` are a single package, decided
once by Q2 — present together, or absent together. There is no partial state.**

- **Recurring automation / data pipeline** → full WAT: `workflows/`, `tools/`, `roles/`, plus
  the package above.
- **One-off script or small tool** → lightweight: skip `workflows/` and skip the package
  above (there is no build state worth tracking). Keep a single role file and a minimal
  `tools/` only if needed. Don't impose overhead the project doesn't earn.
- **Tool with UI** → adapt: WAT for the logic layer, plus whatever the UI requires, plus the
  package above.

In all cases propose this core set and let the user approve or trim:

```
CLAUDE.md         # Trunk file. Preloaded each session. Lean. References everything else.
memory.md         # Self-healing log, written BY the review loop (see below).
PROGRESS.md       # Build state tracker. Only in full-WAT / UI projects (see above).
init.py           # Pre-flight check. Multiplatform (Windows/Linux/Mac). Run before any change.
roles/            # Sequential role instructions (planner / builder / reviewer / scribe*).
tools/convert_to_markdown.py   # Optional. Only if Q4 includes non-Markdown inputs (PDF/Office).
tools/checkpoint.py            # Commits + pushes. Only in full-WAT / UI projects.
.gitignore        # Created at the GitHub step.
```
*`roles/scribe.md` only exists where `PROGRESS.md` exists — see MULTI-ROLE MODEL below.

Explain the role of each file in one line, then wait for approval.

---

## THE MULTI-ROLE MODEL (sequential, not parallel)

One Lead Agent adopts roles **sequentially** by reading the matching file in `roles/`.
Deliberately no parallel subagents — it burns tokens/quota for little gain on personal
projects. Roles:

- **Planner** (`roles/planner.md`) — turns an objective into a task list with success
  criteria.
- **Builder** (`roles/builder.md`) — implements one task at a time using existing tools first.
- **Reviewer** (`roles/reviewer.md`) — runs at the END of each task. Verifies the output
  exists, passes its tests, and meets the expected contract/schema. If it fails, it does NOT
  advance: triggers the correction loop and documents the lesson in `memory.md`. If it
  passes: in full-WAT/UI projects, hands off to Scribe to update `PROGRESS.md`; in
  lightweight projects (no Scribe, no `PROGRESS.md`), Reviewer logs a one-line pass note
  itself and advances — there's nothing to hand off.
- **Scribe** (`roles/scribe.md`) — **only exists in full-WAT / UI projects (see the package
  rule in PHASE 1).** Runs immediately after Reviewer passes a task. Updates
  `PROGRESS.md`'s status markers (`[ ]` → `[~]` → `[x]`) and nothing else — it does not
  verify correctness, that's Reviewer's job.

> Exception (don't apply by default): if a future project has genuinely independent,
> parallelizable work, real subagents may be worth it. The default stays sequential.

---

## PLANNER ROLE REQUIREMENTS (what roles/planner.md must contain)

`roles/planner.md` must instruct the Planner to, after producing the task list and BEFORE
handing off to Builder, self-audit against:

- **Scope** — flag any task not implied by the approved intake/structure. Cut it or ask
  before keeping it.
- **Coverage** — every item from the Requirements Checklist maps to at least one task.
- **Sequencing** — task order respects real dependencies.

Do not proceed to Builder until the audit passes. Log a one-line pass/fail note, and if
something was cut for scope, name what was cut — a bare "pass" is not sufficient.

---

## SCRIBE ROLE REQUIREMENTS (what roles/scribe.md must contain, full-WAT/UI only)

`roles/scribe.md` must instruct the Scribe to, immediately after Reviewer logs a pass:

- Update `PROGRESS.md`: move the completed item's marker to `[x]`, and if the next item was
  `[ ]`, move it to `[~]` to show it's now in progress.
- Do nothing else. Scribe does not judge correctness, does not touch `memory.md`, and does
  not decide what counts as "done" — that was already decided by Reviewer. Scribe's only
  job is keeping `PROGRESS.md` an accurate, current snapshot for the next session to read.

This role file should be short — a few lines. If it starts accumulating logic beyond marker
updates, that logic belongs in Reviewer or Builder, not here.

---

## THE SELF-IMPROVEMENT LOOP (test-driven, not manual notes)

`memory.md` is fed by the review loop, not filled in by hand. On every failure:

1. Reviewer identifies what broke.
2. Builder fixes the tool/code.
3. Reviewer re-verifies the fix.
4. The lesson is appended to `memory.md` in this fixed format:

   ```
   ## [date] — <short title>
   - What failed:
   - Root cause:
   - Fix:
   - How to avoid it next time:
   ```

5. Continue with a more robust system.

**At the start of every session, read `memory.md` first** and apply its lessons. When the
user corrects something or asks you to remember it, append it here in the same format.

---

## PROGRESS.md — BUILD STATUS (full-WAT / UI projects only)

Read at the start of each session, right after `memory.md`. Written only by Scribe, only
when Reviewer has just passed a task (see SCRIBE ROLE REQUIREMENTS above) — never written
by Planner or Builder directly.

States: `[ ]` pending · `[~]` in progress · `[x]` done and verified by Reviewer.

This file answers one question at session start: "where did we leave off?" It is not a
changelog and not a place for reasoning or lessons — that's `memory.md`. Keep each line to
one task, one status marker.

---

## init.py — PRE-FLIGHT CHECK

Create `init.py` (Python, for Windows/notebook portability — not bash). `CLAUDE.md` must
instruct running `python init.py` **before making any change**. It verifies:

- The expected folder/file structure exists.
- Required `.md` files are present and non-empty.
- Project tests (if any) run and pass.

If `init.py` fails, **stop. Do not continue. Ask for help.**

---

## SKILLS & MCP — ON DEMAND ONLY

Before building anything new, check whether an existing Skill or MCP solves the task. Apply
one only if it genuinely helps. If nothing fits, write your own code. Never preload
dependencies "just in case."

---

## NON-MARKDOWN INPUT HANDLING (MarkItDown)

Non-Markdown documents — PDF, DOCX, PPTX, XLSX, or similar office formats — get converted to
Markdown before their content is read, instead of reading raw bytes or writing bespoke
parsing code. Destination is always the agent's context, not a human reader.

**Tool**: [MarkItDown](https://github.com/microsoft/markitdown) (MIT, Microsoft).

**Install on demand, scoped to the formats this project actually uses — never `[all]`:**
```
pip install 'markitdown[pdf,docx,pptx,xlsx]'   # trim to what Q4 actually requires
```

**Wrap it as `tools/convert_to_markdown.py` — never call it inline:**
```python
from markitdown import MarkItDown
from pathlib import Path

md = MarkItDown(enable_plugins=False)
result = md.convert_local(input_path)   # convert_local only — local files, never a URL
Path(output_path).write_text(result.text_content, encoding="utf-8")
```

**Trigger**: any time you need to read a non-Markdown file — declared at Q4, or dropped into
the project later and referenced directly — check for `tools/convert_to_markdown.py` first.
If it doesn't exist yet in this project, create it now (see SKILLS & MCP — ON DEMAND ONLY):
never read the raw file directly instead.

**Known trade-offs to state, not silently absorb:**
- Lossy by design: optimized for LLM ingestion, not visual fidelity.
- I/O runs with the process's own privileges. Only `convert_local()` — never `convert()` on
  an untrusted path or URL.
- Token savings are real but not independently benchmarked; measure on actual files if it
  matters for a decision.

---

## AUTOMATED CHECKPOINTING (full-WAT / UI projects only)

`tools/checkpoint.py` checks for a meaningful change (a real git diff, and `PROGRESS.md`
showing a task that just moved to `[x]`) and, if so, commits and pushes.

Two triggers call it — one script, no duplicated logic:

1. A `Stop` hook, in `.claude/settings.json`, runs it after every turn.
2. A manual `/checkpoint` command runs it on demand.

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [ { "type": "command", "command": "python tools/checkpoint.py" } ] }
    ]
  }
}
```

---

## MODEL HEURISTIC (the user switches manually with `/model`)

Model is set globally via `/model`, not per message. Suggested manual use:

- **Opus** — intake, planning, architecture decisions, hard debugging.
- **Sonnet** — routine code execution, edits, file writing, the bulk of building.
- **Haiku** — trivial/mechanical steps to save quota.

Remind the user to switch when a phase clearly warrants it; don't pretend to switch yourself.

---

## GITHUB — LAST STEP OF INITIALIZATION

Only after the structure exists and the plan is clear, initialize the repo:

- Create `.gitignore` excluding `.env`, credentials, `token.json`, `.tmp/`, and any
  `memory.md` content that holds sensitive data (ask if unsure per project).
- Initialize git, make the first commit (English message), and give the exact commands to
  push to GitHub if the user wants it remote.

---

## CLAUDE.md REQUIREMENTS (the trunk file you will write)

When you scaffold, the `CLAUDE.md` you generate must:

- Stay lean and reference the other files instead of inlining their content.
- Instruct: run `python init.py` before any change; if it fails, stop and ask.
- Instruct: read `memory.md`, then `PROGRESS.md` (if it exists) at session start, in that
  order, before doing anything else.
- Describe the sequential role model and when each role runs, including Scribe if this
  project has a `PROGRESS.md`.
- State the language rule (English artifacts, Spanish communication).
- State the Skills/MCP on-demand rule.
- If `tools/convert_to_markdown.py` exists or Q4 mentioned non-Markdown inputs, state
  explicitly: any non-Markdown file encountered, at any point in the project, gets converted
  via that tool before being read — this must hold every session, not just at scaffold time.
- If this project has `tools/checkpoint.py`, note that it runs automatically via a `Stop`
  hook — commits are not something the agent decides to do mid-task, they happen
  deterministically after Reviewer/Scribe close out a task.
- Link the files together so they reference each other coherently.

---

## START NOW

Begin Phase 0. Ask question 1 in Spanish. Do not create any file until the plan is approved.
