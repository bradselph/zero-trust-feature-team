---
description: Implement one approved step through implementer → test-engineer → step-verifier.
argument-hint: [STEP-id | next]
allowed-tools: Read Grep Glob Edit Write Bash
---

# /feature:implement

Apply one step and verify it independently. Dispatch `@feature-orchestrator`.

The user's target: `$ARGUMENTS` (a specific `STEP-NNNN`, the word `next`, or empty which means `next`)

## Orchestrator, on receipt:

1. **Preflight — the four pre-implementation gates.**

   | Gate | Source of truth | Required value |
   |---|---|---|
   | 1. Task confirmed | `task.json.confirmed_by_user` | `true` |
   | 2. Research complete | `research.json.coverage_pct` | `100.0` (or user accepted partial) |
   | 3. Plan exists | `plan.json` | well-formed |
   | 4. Plan approved | `plan.json.review.status` | `"approved"` |

   If any gate is open, refuse and tell the user exactly which gate is blocked and which command opens it (`/feature:init`, `/feature:research`, `/feature:plan`, `/feature:review`).

2. **Resolve the target step.**
   - If `$ARGUMENTS` is a valid `STEP-NNNN`: load that step.
   - If `$ARGUMENTS` is empty or `"next"`: pick the first step in `plan.steps_summary[]` order whose `status: "planned"` AND whose `depends_on` are all `status: "verified"`.
   - If no eligible step exists: report "all planned steps complete or in progress" and stop. If steps exist with `status: "needs-human"`, surface them with their dissent notes.

3. **Sensitive-path / human-review gate.** If the step has `requires_human_review: true` (and the user has not already provided this approval via `plan.review.user_overrides`):
   - Display the full step (files, anchors, intended_change, risks)
   - Ask: "This step requires human review (sensitive path or breaking change). Approve to proceed? (yes/no, or 'defer' to skip and pick a non-sensitive step)"
   - Do not proceed without explicit `yes`. On `no` or `defer`, mark `step.status: "needs-human"` (or leave `planned` if deferring) and stop.

4. **Dispatch implementer:**

   > `@feature-implementer` — implement step `STEP-NNNN`. Step JSON: `.claude/feature-state/steps/STEP-NNNN.json`. Plan: `.claude/feature-state/plan.json` (note any `user_overrides`). Re-locate anchors before editing. Do not touch files outside `step.files[]`.

5. **On implementer return:**
   - Verify `log/step-STEP-NNNN.md` was written.
   - Verify the step's `status` is now `"implemented"` or `"needs-human"`.
   - If `needs-human`: stop. Report the dissent and ask the user how to proceed.
   - If `implemented`: verify `git diff` only touches files declared in `step.files[]`. If files outside the step's list were modified, this is a scope-creep failure. Revert the implementer's changes (`git checkout -- <unauthorized-files>`), reset step to `planned`, and report the violation. Do not proceed to test-engineer.

6. **Dispatch test-engineer:**

   > `@test-engineer` — write tests for step `STEP-NNNN`. Step log: `log/step-STEP-NNNN.md`. Step spec: `steps/STEP-NNNN.json`. Run Phase A (new tests) and Phase B (co-located existing tests), plus Phase C if the step is `breaking_change: true`.

7. **On test-engineer return:**
   - Verify the step log has a Tests section.
   - Read `test_status` from the log.
   - If `test_status: "regression"`: stop. Report the regression, set step `status: "in-progress"` (or `needs-human` if the regression is intentional and requires user judgment), tell the user the implementer must re-do or the user must accept the regression. Do **not** dispatch step-verifier.
   - If `test_status: "behavior-mismatch"`: same handling — implementer's code does not match the BC's expected; step goes back to `in-progress`.
   - If `test_status: "passing"`, `"manual"`, `"not-applicable"`, or `"no-framework"`: proceed to step 8.

8. **Dispatch step-verifier:**

   > `@step-verifier` — independently verify step `STEP-NNNN`. Verify the four truths: plan-conformance, behavior-conformance, AC-contribution, no-regression. Step JSON: `steps/STEP-NNNN.json`. Step log: `log/step-STEP-NNNN.md`. Research log(s): `log/research-*.md`. Task: `task.json`.

9. **On step-verifier return:**
   - If verdict is `CONFIRM`: step status should now be `"verified"`. Report success.
   - If verdict is `REJECT`: step status is back to `"in-progress"`. Report the rejection reason and the gap the implementer must address.

10. **Report to user:**

    ```
    Step STEP-NNNN: <CONFIRMED | REJECTED | REGRESSION | NEEDS-HUMAN>
      Title: <step.title>
      Files changed: <n>
      Tests: <test_status>
      Verifier: <CONFIRM | REJECT — reason: ...>

      <If CONFIRMED>
      AC contribution: <which ACs this step delivered toward>
      Next: /feature:implement to continue with the next step, or /feature:status.

      <If REJECTED, REGRESSION, or NEEDS-HUMAN>
      Gap to close: <description from verifier or test-engineer or implementer dissent>
      Next: /feature:implement STEP-NNNN to retry, /feature:plan to revise, or address manually.
    ```

## Hard rules

- One step per `/feature:implement` invocation. Do not loop into the next step automatically. The user controls cadence.
- Never skip the test-engineer. Even for trivial-looking steps. If there's no test framework, test-engineer produces `test_status: "manual"` and the step still requires step-verifier approval.
- Never skip the step-verifier. The implementer's claim that the step works is not sufficient.
- Never let the implementer modify files outside `step.files[]`. Revert and reset on violation.
- If any of {implementer, test-engineer, step-verifier} rejects, do not auto-retry. The user decides whether to retry, re-plan, or escalate.
- Never bypass the four pre-implementation gates. Even if the user asks. If gates are open, refuse and explain.
