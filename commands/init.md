---
description: Bootstrap a zero-trust feature implementation. Collect requirements, write task.json, and stage the project for research.
argument-hint: [one-line feature description]
allowed-tools: Read Grep Glob Edit Write Bash
---

# /feature:init

You are starting (or resetting) a zero-trust feature implementation. Dispatch `@feature-orchestrator` to handle this properly.

The user's feature description: `$ARGUMENTS`

## Orchestrator, on receipt:

1. Check whether `.claude/feature-state/` already contains a `task.json`. If yes, ask the user whether this is:
   - a **fresh feature** (archive existing state to `.claude/feature-state/archive/<iso-date>/` and start over), or
   - a **refinement** of the existing task (intake-analyst will update fields and reset `confirmed_by_user` to `false` for re-confirmation).

2. Dispatch:

   > `@intake-analyst` — capture the user's feature request: `$ARGUMENTS`. Interview the user for goals, non-goals, acceptance criteria, constraints, and assumptions. Write `.claude/feature-state/task.json`. Do not infer anything from the codebase.

3. The intake-analyst will likely ask the user follow-up questions in batches. Let it. Do not interrupt or summarize. When the analyst's turn ends, it will print the captured spec and ask the user for explicit confirmation.

4. **Confirmation gate.** When the user replies:
   - If the user says "confirmed" / "yes, that's right" / similar **explicit** confirmation: read `task.json`, set `confirmed_by_user: true` and `confirmation_text: "<verbatim user reply>"`, write back. Do not flip this on anything weaker than an explicit yes.
   - If the user lists corrections: re-dispatch `@intake-analyst` with the corrections; do **not** set `confirmed_by_user`.
   - If the user says nothing committal: do not set the gate. Tell them what you need ("type 'confirmed' to proceed, or list corrections").

5. Once confirmed, report:

   ```
   Feature intake complete.
     Title: <task.title>
     Goals: <count>
     Acceptance criteria: <count>
     Open questions: <count> (<n> block research, <m> block plan)
     Assumptions to validate: <count>
   
   task.confirmed_by_user: true ✓
   
   Next: /feature:research to begin codebase research.
   ```

6. If the intake-analyst returns without writing `task.json`, or with required fields missing, redispatch with a note about what was missing. Do not proceed to research with a malformed task.

## Hard rules

- Do not dispatch `@project-researcher` from this command. Research is a separate phase, gated on `confirmed_by_user`.
- Do not flip `confirmed_by_user` to `true` without an explicit user reply that is unambiguously a confirmation.
- Do not silently overwrite an existing `task.json`.
