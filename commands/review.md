---
description: Independent review of the plan, plus user approval. The final gate before any code is written.
allowed-tools: Read Grep Glob Edit Write
---

# /feature:review

Run the plan-critic and present the plan to the user for explicit approval. Dispatch `@feature-orchestrator`.

## Orchestrator, on receipt:

1. **Preflight.** Confirm:
   - `.claude/feature-state/plan.json` exists
   - `.claude/feature-state/steps/STEP-*.json` exists for every step in `plan.steps_summary[]`
   - `.claude/feature-state/task.json` and `.claude/feature-state/research.json` exist (the critic needs them)

   If any are missing, stop and tell the user to run `/feature:plan` first.

2. **Dispatch:**

   > `@plan-critic` -- independently review `.claude/feature-state/plan.json` against `task.json` and `research.json`. Verify the four truths: AC coverage, evidence anchors, constraint adherence, ordering soundness. Write your verdict to `plan.json.review.critic_verdict` and any recommendations to `plan.json.review.critic_recommendations`. Do not modify steps or the task.

3. **On critic return:**
   - Read `plan.json.review.critic_verdict`.
   - Read `plan.json.review.critic_recommendations`.

4. **Branch on verdict:**

   ### CONFIRM
   
   Display the plan + critic recommendations to the user. Format:

   ```
   +- Plan Review -----------------------------------------
   |
   |  Critic verdict: CONFIRM
   |
   |  Plan summary:
   |    Total steps:           <n>
   |    Files to be modified:  <list>
   |    Files to be created:   <list>
   |    ACs covered:           <m>/<m>
   |    Sensitive-path steps:  <list> (require human review)
   |    Breaking-change steps: <list>
   |
   |  Risks logged: <r>
   |  Deferred (out-of-scope decisions): <e>
   |
   |  Critic recommendations (non-blocking):
   |    REC-1  <description>  -> <linked step>
   |    REC-2  ...
   |
   |  Steps in execution order:
   |    STEP-0001  [<kind>]  <files>  -- <title>
   |    STEP-0002  ...
   |
   \---------------------------------------------------------

   Approval gate. To proceed to /feature:implement, reply with one of:
     "approved"                 -> orchestrator marks plan approved as-is
     "approved with: <list>"    -> orchestrator records overrides and marks approved
     "changes: <list>"          -> orchestrator instructs planner to revise; you'll re-run /feature:review after
     "reject"                   -> plan archived, you can re-init or re-plan
   ```

   Wait for the user's reply.

   - **"approved"**: set `plan.review.status: "approved"`, `plan.review.user_approved_at: "<iso8601>"`, `plan.review.user_approval_text: "<verbatim reply>"`. Print:

     ```
     Plan approved. plan.review.status -> "approved".
     Next: /feature:implement to apply the first step (or /feature:implement STEP-NNNN to jump).
     ```

   - **"approved with: <list>"**: same as above, but additionally record each override in `plan.review.user_overrides[]`. The orchestrator passes overrides to the implementer at dispatch time so the implementer can honor them.

   - **"changes: <list>"**: do **not** flip `review.status`. Re-dispatch `@feature-planner` with the user's requested changes, then loop back to step 2 (review again with critic + user). The critic re-runs every time the plan changes.

   - **"reject"**: archive `plan.json` and `steps/` to `plan-<iso>.json` and `steps-<iso>/`. Set `plan.review.status: "rejected"`. Print: `Plan rejected. Run /feature:init to start over or /feature:plan to draft a new plan from existing research.`

   ### REJECT

   The critic found a blocking gap. Display:

   ```
   +- Plan Review -----------------------------------------
   |
   |  Critic verdict: REJECT
   |  Reason: <one of: ac-uncovered | constraint-dropped | fabricated-anchor | ...>
   |
   |  Blocking gaps:
   |    GAP-1  <description>
   |           Evidence: <file:line -- substring>
   |    GAP-2  ...
   |
   |  Recommended action:
   |    Re-run /feature:plan after the planner addresses these gaps,
   |    OR explicitly accept-with-overrides if you judge a gap acceptable.
   |
   \---------------------------------------------------------

   Reply with one of:
     "re-plan"                  -> orchestrator re-dispatches the planner with the critic's gaps
     "accept gaps: <list>"      -> orchestrator records the user's override, sets status approved, proceeds
     "abort"                    -> plan archived
   ```

   Wait for the reply. Handle as in the CONFIRM branch's analogous options.

## Hard rules

- Never flip `plan.review.status` to `"approved"` without an explicit user reply that contains the word `"approved"` (or equivalent unambiguous confirmation). The orchestrator does not infer approval.
- Never proceed to `/feature:implement` from this command. Implementation is a separate command and a separate gate.
- Never edit step files from here. If the critic finds an issue with a step, the planner re-runs.
- Never hide critic recommendations from the user. Even non-blocking recs must be displayed so the user can accept-with-overrides knowingly.

## When the critic and user disagree

- Critic CONFIRM, user REJECT (says "reject" or "changes"): the user wins. Plan archives or re-plans.
- Critic REJECT, user "accept gaps": the user wins, but the override is recorded in `plan.review.user_overrides[]`. The implementer and step-verifier can see the override and treat the gap as accepted scope rather than failing on it.
- Critic REJECT, user "re-plan": planner re-runs. New plan must address the gaps; if it does not, the next critic pass will REJECT again.
