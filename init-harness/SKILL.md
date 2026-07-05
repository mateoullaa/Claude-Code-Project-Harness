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

- **Recurring automation / data pipeline** → full WAT: `workflows/`, `tools/`, `roles/`.
- **One-off script or small tool** → lightweight: skip `workflows/`, keep a single role file
  and a minimal `tools/` only if needed. Don't impose overhead the project doesn't earn.
- **Tool with UI** → adapt: WAT for the logic layer, plus whatever the UI requires.

In all cases propose this core set and let the user approve or trim:

```
CLAUDE.md         # Trunk file. Preloaded each session. Lean. References everything else.
memory.md         # Self-healing log, written BY the review loop (see below).
init.py           # Pre-flight check. Multiplatform (Windows/Linux/Mac). Run before any change.
roles/            # Sequential role instructions (planner / builder / reviewer).
tools/convert_to_markdown.py   # Optional. Only if Q4 includes non-Markdown inputs (PDF/Office).
.gitignore        # Created at the GitHub step.
```

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
  passes, logs a one-line note and advances.

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

If the project's inputs (Q4) include non-Markdown documents — PDF, DOCX, PPTX, XLSX, or
similar office formats — the Builder converts them to Markdown before ingesting their
content, instead of reading raw bytes or writing bespoke parsing code per project.
Destination is always the agent's context, not a human reader.

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

**Trigger**: only when a specific task requires reading a non-Markdown input file — never a
blanket filesystem scan during `init.py`.

**Known trade-offs to state, not silently absorb:**
- Lossy by design: optimized for LLM ingestion, not visual fidelity.
- I/O runs with the process's own privileges. Only `convert_local()` — never `convert()` on
  an untrusted path or URL.
- Token savings are real but not independently benchmarked; measure on actual files if it
  matters for a decision.

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
- Instruct: read `memory.md` at session start and apply its lessons.
- Describe the sequential role model and when each role runs.
- State the language rule (English artifacts, Spanish communication).
- State the Skills/MCP on-demand rule.
- Link the files together so they reference each other coherently.

---

## START NOW

Begin Phase 0. Ask question 1 in Spanish. Do not create any file until the plan is approved.
