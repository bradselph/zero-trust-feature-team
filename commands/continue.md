---
description: Resume a paused research run from the last STATUS: PARTIAL marker.
allowed-tools: Read Grep Glob Edit Write Bash
---

# /feature:continue

Resume a paused research area where it stopped. Dispatch `@feature-orchestrator`.

## Orchestrator, on receipt:

1. Read `.claude/feature-state/research.json` and find an area with `status: "partial"` (there should be at most one; if more than one, something went wrong — report it).

2. If no `partial` area exists, this command is a no-op. Check whether there are `"not-started"` areas:
   - If yes, the user should run `/feature:research` instead. Tell them.
   - If no, research is complete. Tell them to run `/feature:plan` (or `/feature:status` for a snapshot).

3. Otherwise, dispatch:

   > `@project-researcher` — resume research of area `<id>` at paths `<glob list>`. Resume marker: `<file:line>`. Reason for previous pause: `<reason>`. Task: `.claude/feature-state/task.json`. Prior research dir: `.claude/feature-state/log/`.

4. On researcher return, same verification as `/feature:research` step 4. After that, this command hands control back — it does not loop into `/feature:research`. The user can chain commands explicitly if they want to.

## Why a separate continue command

The research phase is the only chunked, multi-turn phase that can pause mid-area. The implement phase is one step per turn by design — there is no "partial step", so no resume command is needed there. If a step gets `needs-human`, the user re-runs `/feature:implement STEP-NNNN` after addressing the issue; that's a re-attempt, not a resume.
