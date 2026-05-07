---
description: Show current feature progress, pre-implementation gate state, step status, and next recommended action.
allowed-tools: Read Grep Glob
---

# /feature:status

Read-only snapshot. Dispatch `@feature-orchestrator` but instruct it to dispatch **no specialists** -- pure state read.

## Orchestrator, on receipt:

1. **Existence check.** For each file, note whether it exists:
   - `.claude/feature-state/task.json`
   - `.claude/feature-state/research.json`
   - `.claude/feature-state/plan.json`
   - `.claude/feature-state/steps/STEP-*.json` (count files)
   - `.claude/feature-state/log/research-*.md` (count files)
   - `.claude/feature-state/log/step-*.md` (count files)

   If the state directory is entirely missing or empty, report "No feature initialized" and tell the user to run `/feature:init`.

2. **Pre-implementation gates.** From `task.json`, `research.json`, `plan.json`:

   ```
   Gate 1 -- Task confirmed:    <[OK] | [X] -- task.confirmed_by_user is false | [X] -- task.json missing>
   Gate 2 -- Research complete: <[OK] | [X] -- coverage_pct = X | [X] -- research.json missing>
   Gate 3 -- Plan exists:       <[OK] | [X] -- plan.json missing>
   Gate 4 -- Plan approved:     <[OK] | [X] -- review.status = draft | [X] -- review.status = rejected>
   ```

3. **Task snapshot** (if `task.json` exists):
   - Title
   - Goals count
   - ACs count
   - Open questions count, broken down by `blocks_phase`
   - Assumptions count, broken down by validation status

4. **Research snapshot** (if `research.json` exists):
   - `areas_covered / areas_total` and `coverage_pct`
   - Count of areas in each status: `not-started`, `partial`, `covered`, `research-failed`
   - List any areas with `status: "partial"` and their resume markers

5. **Plan snapshot** (if `plan.json` exists):
   - Total steps
   - Steps requiring human review
   - Steps marked breaking-change
   - Risk count
   - Deferred items count
   - `review.status` (`draft` | `approved` | `rejected`) and `critic_verdict` (`confirm` | `reject` | `null`)

6. **Step snapshot** (enumerate `steps/STEP-*.json`):
   - Count by status: `planned`, `in-progress`, `implemented`, `verified`, `needs-human`, `wontfix`, `deferred`
   - List any `needs-human` steps with their `implementer_dissent`
   - Verified count vs total -> completion %

7. **AC coverage snapshot** (cross-reference task ACs against verified steps):
   - For each AC, compute: list of steps that cover it, count of those steps that are `verified`
   - An AC is "delivered" only when **all** steps covering it are `verified`
   - Report: `ACs delivered: <d>/<total>`

8. **Determine next recommended action** using this decision tree:

   ```
   No task.json?                          -> /feature:init
   task.confirmed_by_user = false?        -> tell user to confirm captured task (see /feature:init output)
   No research.json or coverage < 100%?   -> /feature:research (or /feature:continue if there's a partial area)
   No plan.json?                          -> /feature:plan
   plan.review.status = draft?            -> /feature:review
   plan.review.status = rejected?         -> /feature:plan (re-plan) or /feature:init (restart)
   Any step status = needs-human?         -> resolve dissent, then /feature:implement STEP-NNNN
   Any step status = planned/in-progress? -> /feature:implement
   All verified?                          -> /feature:summary
   ```

9. **Output format:**

   ```
   +- Feature Status --------------------------------------
   |
   |  Task: <title or "(none)">
   |
   |  Gates:
   |    1. Task confirmed:    <[OK] | [X]>
   |    2. Research complete: <[OK] | [X]>
   |    3. Plan exists:       <[OK] | [X]>
   |    4. Plan approved:     <[OK] | [X]>
   |
   |  Research: <Y>/<X> areas * <pct>%
   |    +-- covered:        <n>
   |    +-- partial:        <n>  (resume: <area>:<file>:<line>)
   |    +-- not-started:    <n>
   |    \-- research-failed:<n>
   |
   |  Plan: <n> steps * review.status = <draft | approved | rejected>
   |    Critic verdict:        <confirm | reject | n/a>
   |    Requires human review: <c>
   |    Breaking-change steps: <b>
   |    Risks logged:          <r>
   |
   |  Steps:
   |    planned:      <n>
   |    in-progress:  <n>
   |    implemented:  <n>  (awaiting step-verifier)
   |    verified:     <n>  [OK]
   |    needs-human:  <n>  WARN:
   |    wontfix:      <n>
   |    deferred:     <n>
   |
   |  ACs delivered: <d>/<total>
   |    AC-1: <title>  -> <verified | partial: STEP-NNNN open>
   |    AC-2: ...
   |
   |  Next: <recommended command>
   |
   \---------------------------------------------------------
   ```

   If any `needs-human` steps exist, list them explicitly below the box with their dissent notes. These are blocked on the user's judgment and should not get buried.

## Hard rules

- This command never modifies state. `allowed-tools` excludes `Edit`, `Write`, `Bash`.
- This command never dispatches specialist agents. The orchestrator reads state directly.
- If state files are malformed (invalid JSON, missing required fields), report the specific file and field -- do not silently skip. Corrupt state is itself a status worth surfacing.
- If counts across `research.json`, `plan.json`, `steps/`, and `log/` disagree, report the discrepancy. Do not pick a winner -- let the user investigate.
