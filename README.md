# Zero-Trust Feature Team

A Claude Code plugin that turns a one-line feature request into a fully-tracked implementation pipeline: **intake -> research -> plan -> review -> implement -> test -> verify**, with persistent state that survives `/clear` and context resets.

Where the [Zero-Trust Audit Team](https://github.com/bradselph/zero-trust-audit-team) finds and fixes existing bugs, this plugin builds new features the same way: every claim backed by evidence, every step independently verified, no code written until the user explicitly approves the plan.

## Install

```
/plugin install https://github.com/bradselph/zero-trust-feature-team
```

Or load locally for testing:

```bash
git clone https://github.com/bradselph/zero-trust-feature-team
claude --plugin-dir ./zero-trust-feature-team
```

## Quickstart

```
/feature:init "add server-side search to the documents list"
/feature:research
/feature:plan
/feature:review
/feature:implement
/feature:summary
```

## Commands

| Command | Description |
|---|---|
| `/feature:init [description]` | Capture requirements via the intake-analyst; user must confirm before anything else runs |
| `/feature:research` | Research the codebase area-by-area (stops at `STATUS: PARTIAL`) |
| `/feature:continue` | Resume a paused research run at the last resume marker |
| `/feature:plan` | Convert task + research into an ordered, atomic step plan |
| `/feature:review` | Plan-critic pass plus user approval gate -- no code is written until this approves |
| `/feature:implement [STEP-id]` | Implement -> test -> verify one step (one step per invocation) |
| `/feature:status` | Snapshot: gates open/closed, research coverage, step status, AC delivery, next action |
| `/feature:summary` | Final report (only valid after every step is in a terminal state) |

## The team

Eight specialized agents coordinate through a shared state ledger in `.claude/feature-state/`. Each has a narrow, evidence-enforced role:

| Agent | Role | Writes code? |
|---|---|---|
| `feature-orchestrator` | Coordinator -- dispatches all other agents, owns state, enforces gates | State files only |
| `intake-analyst` | Adversarial requirements interview, writes `task.json` | No |
| `project-researcher` | Zero-trust per-area researcher -- one area per invocation, explicit evidence required | No |
| `feature-planner` | Decomposes the feature into atomic, ordered, verifiable steps with anchors | No |
| `plan-critic` | Independent review of the plan -- confirms or rejects with concrete gaps | No |
| `feature-implementer` | Applies one step at a time -- minimal change, no scope creep | Yes |
| `test-engineer` | Writes tests for each step's behavior_checks; runs existing tests | Tests only |
| `step-verifier` | Independent re-trace of the four truths -- confirms or rejects with evidence | No |

## The flow

```
/feature:init      ->  capture requirements, user confirms task.json
/feature:research  ->  project-researcher covers areas one-by-one, writes log/research-*.md
/feature:plan      ->  feature-planner builds plan.json + steps/STEP-NNNN.json
/feature:review    ->  plan-critic verdict + user approval flips review.status to "approved"
/feature:implement ->  feature-implementer -> test-engineer -> step-verifier (one step per run)
/feature:summary   ->  final report: AC delivery, steps executed, residual risks
```

Each phase is a separate command. Nothing happens automatically between phases -- you stay in control of when to move forward.

## The four pre-implementation gates

No `feature-implementer` dispatch is allowed until **all** four gates are satisfied:

1. **Task confirmed** -- `task.json.confirmed_by_user == true`. Set only after the intake-analyst presents the captured spec and the user replies with explicit confirmation.
2. **Research complete** -- every research area is `covered` (or explicitly accepted as partial by the user).
3. **Plan exists** -- `plan.json` with one `steps/STEP-NNNN.json` per step.
4. **Plan approved** -- `plan.review.status == "approved"`. Set only after the plan-critic's verdict and the user replying with explicit approval.

These gates are why the user is in the loop *before* code is written, not after it lands as a surprise diff.

## State layout

All state lives in `.claude/feature-state/` in your project as plain JSON and Markdown -- diffable, reviewable, readable without tooling.

```
.claude/feature-state/
+--- task.json                  Confirmed feature spec: goals, ACs, constraints, assumptions
+--- research.json              Per-area research index: status, integration points, gaps, verdicts
+--- plan.json                  Ordered plan summary, risks, deferred items, review state
+--- steps/                     STEP-0001.json, STEP-0002.json, ... -- one file per step
+--- log/
|   +--- research-<area>.md     Per-area execution traces from project-researcher
|   \--- step-STEP-NNNN.md      Step record: edits + tests + step-verifier verdict
\--- README.md                  Schema reference for every state file
```

The `feature-state/README.md` schema reference is included in this plugin for reference. The orchestrator writes it to your project during `/feature:init`.

## Design choices

**Why eight agents.** Each agent has one job and a narrow tool set. The implementer cannot research, the researcher cannot plan, the planner cannot edit code, the verifier cannot approve its own team's work. Separation makes scope creep mechanically harder.

**Why one area per research invocation.** Researching a whole codebase in one shot loses fidelity. One area at a time produces a per-area `log/research-<area>.md` with anchors and traces -- survives `/clear`, resumable across sessions on any repo size.

**Why one step per implement invocation.** Batching multi-step changes is where regressions hide. Each step gets its own complete trace: anchor re-location -> edit -> static checks -> tests -> independent re-verification.

**Why the planner cannot edit code.** The planner's incentive to "just do it" while planning is the most common source of partial implementations and untraceable diffs. Read-only by construction.

**Why the critic cannot edit the plan.** The critic's job is to find gaps. A critic that can patch the plan it reviews will rationalize gaps instead of flagging them. The critic produces recommendations; the planner must re-run to apply them.

**Why re-verification is a separate agent.** The agent that applied the step is the wrong agent to certify it. The `step-verifier` starts from a clean context and must produce an independent trace against four truths: plan-conformance, behavior-conformance, AC-contribution, no-regression.

**Why steps must declare anti-checks.** "Diff confined to anchor region" is not aspirational -- it's a check the verifier runs. Catches scope creep mechanically rather than through diligence.

## Customizing

**Constraint policy** -- edit `agents/feature-planner.md`, the cross-cutting risks step (sensitive paths, breaking-change rules).

**Per-language test conventions** -- the test-engineer auto-detects framework. If your project has unusual test infrastructure, drop a hint in `task.constraints.must_use` (e.g., `"the existing custom test harness in scripts/test.sh"`) -- the test-engineer reads constraints first.

**Sensitive path overrides** -- add additional sensitive globs in your task's `constraints.security` field. The planner promotes any step touching them to `requires_human_review: true`.

**Pre-existing research notes** -- drop Markdown into `.claude/feature-state/log/research-<area>.md` matching the format in `feature-state/README.md` and add the area to `research.json` with `status: "covered"`. The planner reads both as first-class evidence.

## When not to use this

- **Trivial single-file edits**: overkill -- use plain Claude Code.
- **Greenfield projects with no existing patterns**: the researcher's value is on accumulated code with conventions to mirror.
- **Spike/throwaway prototypes**: the gates are intentional. If you want fast and ungated, stay in plain Claude Code.
- **You already have a written design doc and just want it executed**: you can drop a research summary in by hand and skip `/feature:research`, but the planner will still expect anchors that resolve on disk.

## Companion: the audit plugin

This plugin builds new features. Its sibling, [Zero-Trust Audit Team](https://github.com/bradselph/zero-trust-audit-team), audits and fixes existing code. They share design DNA: chunked invocations, evidence-backed claims, persistent state, independent verification. Use them together -- audit before adding to fragile areas, feature after to extend solid ones.
