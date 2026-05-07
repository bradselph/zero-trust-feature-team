---
description: Produce the final feature implementation report. Only valid when every step is in a terminal state and every AC is delivered or explicitly accepted as undelivered.
allowed-tools: Read Grep Glob Write
---

# /feature:summary

Final report generation. Dispatch `@feature-orchestrator`.

## Orchestrator, on receipt:

### 1. Preflight -- strict

The summary is premature if **any** of these are true:

- Any step in `steps/` has `status` in: `planned`, `in-progress`, `implemented`
  - (`implemented` means awaiting step-verifier -- not a terminal state)
- `plan.json` does not exist (nothing was planned)
- `plan.review.status` is not `"approved"` (the plan was never accepted)

Terminal step statuses are: `verified`, `wontfix`, `needs-human`, `deferred`. A step in `needs-human` is acceptable in the summary if the user explicitly chose to ship without resolving it (that decision must be recorded in the user's reply).

If preflight fails, **refuse** to produce the summary. Output:

```
Feature summary is premature. Blocking items:
  - <n> steps in non-terminal status:
      planned:      STEP-NNNN, ...
      in-progress:  ...
      implemented:  ... (awaiting step-verifier)

Next: /feature:implement to close each gap, or /feature:status for a full snapshot.
```

Do not write a partial report. Do not proceed.

### 2. Write the report

Write `.claude/feature-state/FINAL_REPORT.md` with the following sections.

#### Section A -- Feature

- Title (from `task.json.title`)
- Summary (from `task.json.summary`)
- Started (from `task.created_at`)
- Completed (`<iso8601>` now)
- User's verbatim ask (from `task.user_request_verbatim`)

#### Section B -- Acceptance Criteria

For every AC in `task.acceptance_criteria[]`:

- The AC (verbatim given/when/then)
- Status: `delivered` (all covering steps `verified`) | `partial` (some steps verified, others terminal-non-verified) | `undelivered` (no covering steps reached `verified`)
- Steps that contributed: `STEP-NNNN (verified)`, `STEP-MMMM (needs-human)`, etc.
- For `partial` or `undelivered`, the gap (which steps fell short and why)

#### Section C -- Steps Executed

Table format, one row per step:

| Step | Order | Kind | Title | Files | ACs | Status | Verifier verdict |
|---|---|---|---|---|---|---|---|
| STEP-0001 | 1 | add-symbol | ... | src/foo.ts | AC-1 | verified | CONFIRM |
| ... | | | | | | | |

Sourced from `steps/*.json` and `log/step-*.md`.

#### Section D -- Research Coverage

Brief summary of research that backed the plan:

- Areas researched: <list with file counts>
- Integration points found: <count>
- Reusable primitives reused by the implementation: <list>
- Gaps closed by new code: <list>
- Assumption verdicts (CONFIRMED/REFUTED counts; surfaced PARTIAL/UNVERIFIABLE in Section F)

#### Section E -- Plan Decisions

- The critic's recommendations (`plan.review.critic_recommendations`)
- The user's overrides (`plan.review.user_overrides`) -- these are the cases where the user chose to ship past a critic concern
- Steps deferred at planning (`plan.deferred[]`)

#### Section F -- Residual Risks and Open Items

For every entry in `plan.risks[]`:
- Description, likelihood, impact, mitigation
- Whether the linked steps are now `verified` (risk closed) or non-terminal-but-shipped (risk accepted)

For every step with terminal status `needs-human`, `wontfix`, or `deferred`:
- The step
- Reason / dissent
- What the user accepted

For every research assumption that came back `partial` or `unverifiable`:
- The assumption
- Why it could not be verified
- Whether the implementation accommodated it

#### Section G -- Tests Added

For every step that has a Tests section in its log:
- Test files added or modified
- Test names (with their STEP/BC ids)
- `test_status` per step (passing / manual / not-applicable)

This section is the test surface a future maintainer can run against this feature.

#### Section H -- Metadata

- task.created_at
- plan.built_at
- plan.review.user_approved_at
- Final report timestamp
- Tool versions (if known)
- Agent set used (read from `.claude/agents/` -- list each agent file + its last-modified date)
- Steps total: <n>; verified: <v>; non-verified terminal: <t>
- ACs total: <n>; delivered: <d>; partial: <p>; undelivered: <u>

This metadata is what makes the report reproducible. A future engineer can reference it to know what baseline to compare against.

### 3. Output to chat

```
Final report written: .claude/feature-state/FINAL_REPORT.md
  Steps: <v>/<total> verified  (<t> terminal non-verified: <breakdown>)
  ACs delivered: <d>/<total>  (<p> partial, <u> undelivered)
  Tests added: <n>
  Residual risks: <r> (<closed> closed, <accepted> accepted)

The feature implementation is complete.
```

Do not append summarizing commentary. The report itself is the artifact; the chat message points to it.

## Hard rules

- Never produce a partial summary. If preflight fails, refuse.
- Never fabricate entries. Every row in the steps table comes from `steps/*.json` and `log/step-*.md`. Every AC entry comes from `task.json`. Every test entry comes from a step log section.
- Never reclassify ACs in the summary. If an AC is `undelivered`, it stays `undelivered` -- the summary is a factual reconstruction, not a re-grading.
- Never omit `needs-human`, `wontfix`, or `deferred` items to make the report look cleaner. Those are the whole point -- they tell future engineers what was consciously left undone.
- Do not write the report to `/mnt/user-data/outputs/` or anywhere outside `.claude/feature-state/`. The report belongs to the feature state, versioned with the rest of it.
