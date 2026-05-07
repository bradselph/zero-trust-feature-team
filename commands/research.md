---
description: "Research the codebase area-by-area against the confirmed task. One area per turn, stops on STATUS: PARTIAL."
allowed-tools: Read Grep Glob Edit Write Bash
---

# /feature:research

Execute the chunked research phase. Dispatch `@feature-orchestrator`.

## Orchestrator, on receipt:

1. **Preflight.** Confirm:
   - `.claude/feature-state/task.json` exists AND `confirmed_by_user == true`. If not, stop and tell the user to run `/feature:init` and explicitly confirm the captured task.
   - `.claude/feature-state/research.json` exists. If not, **build it**:
     - Read `task.json`. From the task's goals, ACs, constraints, and assumption tags, derive the list of research areas. Common areas: entry points, data layer, auth, routing, the existing analog of the feature, the test infrastructure, the build/deploy pipeline, the styling/UI conventions (for UI features).
     - Each area gets an `id` (kebab-case, e.g., `routing`, `data-model`, `auth`), a `paths` glob list, and a `priority` (security-sensitive first, then anything that affects core integration points, then peripherals).
     - Initialize `research.json` with every area at `status: "not-started"`.

2. **Find the next area.** From `research.json.areas`, pick the first whose status is `"not-started"` or `"partial"`. Respect priority order.
   - If none exists, go to step 5.
   - If the area's status is `"partial"`, the resume marker is in `research.json.areas[<id>].resume_marker`.

3. **Dispatch the researcher.** Pass the area id, paths, resume marker (if any), and references to the task and prior research:

   > `@project-researcher` -- research area `<id>` at paths `<glob list>`. Resume marker `<file:line>` if continuing. Task: `.claude/feature-state/task.json`. Prior research dir: `.claude/feature-state/log/`.

4. **On researcher return:**
   - Verify the researcher's output ends with `STATUS: COMPLETE` or `STATUS: PARTIAL`.
   - Verify `research.json` was updated for this area.
   - Verify the per-area log was written: `log/research-<id>.md`.
   - Verify any claimed `assumption_verdicts` and `open_question_answers` are present and reference real evidence (`<file>:<line> -- <substring>`).
   - If any of those verifications fail, reject and redispatch with a note about what was missing.

   Then:
   - If `STATUS: COMPLETE`: report progress and **loop back to step 2** for the next area. Continue until token budget is tight or all areas done.
   - If `STATUS: PARTIAL`: report the resume marker and **stop**. The user will run `/feature:continue` next turn.

5. **All areas covered.** Report:

   ```
   Research complete.
     Areas covered: <n>/<n> * 100%
     Files inspected: <total>  *  Lines: <total>
     Assumptions resolved: <c confirmed> <r refuted> <p partial> <u unverifiable>
     Open questions answered: <a>/<b>
     Integration points identified: <i>
     Reusable primitives: <p>
     Gaps to fill via the feature: <g>
   
   Next: /feature:plan to build the implementation plan.
   ```

## Loop discipline

The orchestrator is allowed to dispatch the researcher multiple times in a single `/feature:research` turn, one area at a time, as long as each invocation returns promptly. If the orchestrator's own output is getting long, it should stop after the current area and tell the user to run `/feature:research` again to continue.

Never dispatch the researcher on multiple areas simultaneously. The researcher is designed to process exactly one area per invocation; parallel dispatch defeats the chunking protocol.

## Failure handling

- **Researcher returns without a STATUS marker**: redispatch with "your previous output lacked a STATUS marker; please emit one".
- **Researcher claims `STATUS: COMPLETE` but `research.json` wasn't updated**: redispatch.
- **Researcher returns evidence-less claims** (no anchors, no line numbers): redispatch with the instruction to back claims with the evidence block specified in the agent prompt.
- **Same area fails twice**: mark it `status: "research-failed"` in `research.json` with the failure reason, skip to next, and flag in the user-facing summary. The planner will treat research-failed areas as risk items.
