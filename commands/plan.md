---
description: Build the ordered implementation plan from the confirmed task and complete research.
allowed-tools: Read Grep Glob Edit Write
---

# /feature:plan

Convert task + research into a sequenced, surgical step plan. Dispatch `@feature-orchestrator`.

## Orchestrator, on receipt:

1. **Preflight.** Confirm:
   - `.claude/feature-state/task.json` exists AND `confirmed_by_user == true`
   - `.claude/feature-state/research.json` exists AND `coverage_pct == 100.0` (every area `covered` or `research-failed`)
   - At least one `log/research-*.md` file exists with content

   If files remain unresearched, warn the user:

   ```
   Warning: <n> research areas not covered. Planning now will rely on incomplete codebase knowledge.
   Open: <area-id1>, <area-id2>, ...

   Proceed anyway? (respond 'yes' to plan against incomplete research, or run /feature:research first)
   ```

   Default to waiting for user confirmation. If they say `yes`, continue but record `plan.notes.partial_research: true` and treat unresearched areas as risk items in the plan.

2. **Check for prior plan.** If `plan.json` exists:
   - If no steps have started (every step `status: "planned"`): archive the previous `plan.json` to `plan-<iso-date>.json` and `steps/` to `steps-<iso-date>/`, then build fresh.
   - If steps have started (any in `in-progress | implemented | verified`): preserve those steps, re-plan only the remaining ones, and set `plan.notes.replanned_at: "<iso8601>"`.

3. **Dispatch:**

   > `@feature-planner` — build the implementation plan from `task.json` and `research.json`. Cite research evidence by file/line/anchor. Decompose into atomic steps with verification specs. Write `.claude/feature-state/plan.json` and one `steps/STEP-NNNN.json` per step.

4. **On planner return:**
   - Verify `plan.json` was written and conforms to schema.
   - Verify one `steps/STEP-NNNN.json` exists per step in `plan.steps_summary[]`.
   - Verify every step has `files[]`, `verification`, and at least one `acceptance_criteria_covered[]` entry.
   - Verify every AC in `task.json.acceptance_criteria[]` appears in at least one step's `acceptance_criteria_covered[]` (otherwise the planner missed coverage).
   - Verify `plan.review.status == "draft"` (the planner does not flip the review status).

   If any verification fails, reject and redispatch with the specific gap.

5. **Report:**

   ```
   Plan drafted.

     Total steps: <n>
       Atomic (no deps):       <a>
       Sequenced (with deps):  <b>
       Requires human review:  <c>
       Breaking changes:       <d>
     ACs covered:               <m>/<m>
     Deferred (out-of-scope):   <e>
     Risks logged:              <r>

   Top 5 steps by order:
     STEP-0001  [<kind>]  <files>  — <title>
     STEP-0002  [<kind>]  <files>  — <title>
     ...

   plan.review.status: draft

   Next: /feature:review for the plan-critic's pass and your final approval. No code is written until you explicitly approve.
   ```

## Edge cases

- **Zero steps**: the task is already implemented or the planner found no work to do. Report that with the rationale and suggest closing the task with `/feature:summary` (which will note "no implementation needed").
- **All steps require human review**: the feature is high-risk (sensitive paths, breaking changes throughout). Surface this clearly and recommend the user take the review step seriously.
- **Plan reviews itself as not implementable** (a step has anchor not found, contracts conflict, etc.): the planner will surface this. Do not proceed; ask the user how to handle (re-research, change task, etc.).
