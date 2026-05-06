# Feature State — Schema Reference

All agents read from and write to this directory. Every file is plain JSON or Markdown — diffable, reviewable, and persistent across `/clear`.

> **Schema policy.** The templates below are the *only* legal shapes. Field names, casing, and enums are normative. Agents must copy these templates verbatim and fill in values — they must not invent fields, rename fields, or change casing.

---

## `task.json`

Captured by `intake-analyst`. The user must explicitly confirm before any later phase runs.

```json
{
  "created_at": "<iso8601>",
  "updated_at": "<iso8601>",
  "title": "Add server-side search to the documents listing",
  "summary": "Users currently scroll through hundreds of documents. We need server-side full-text search with filters for owner and date.",
  "user_request_verbatim": "add search to the docs page, server side",
  "goals": [
    "Users can search documents by title and body",
    "Users can filter by owner and date range"
  ],
  "non_goals": [
    "Faceted UI (out of scope for v1)",
    "Search across other resource types"
  ],
  "acceptance_criteria": [
    {
      "id": "AC-1",
      "given": "a user with at least one document containing 'invoice' in the body",
      "when": "the user submits the search query 'invoice'",
      "then": "the response contains that document and no documents lacking the term"
    }
  ],
  "constraints": {
    "must_use": ["the existing /api/documents endpoint pattern", "Postgres full-text search"],
    "must_not_use": ["external search services (no Algolia, no Elasticsearch)"],
    "performance": "p95 < 200ms for queries against ≤10k documents",
    "compatibility": "must work in supported browsers from the existing list",
    "security": "results must respect existing per-user document ACLs",
    "ui_ux": null
  },
  "users": ["end users with at least one document"],
  "open_questions": [
    { "id": "Q-1", "question": "Stemming or exact-match?", "blocks_phase": "plan" }
  ],
  "assumptions": [
    { "id": "A-1", "assumption": "documents.body is already a tsvector or can be indexed with one", "must_validate_in": "research" }
  ],
  "confirmed_by_user": false,
  "confirmation_text": null
}
```

**Lowercase enum values only.** `confirmed_by_user` is `false` until the user types an explicit confirmation; the orchestrator (not the intake-analyst) flips it to `true`.

---

## `research.json`

Top-level shape is an object. Per-area entries are keyed by `area_id` under `areas`. Do not flatten to `[ {id, status}, ... ]`.

```json
{
  "built_at": "<iso8601>",
  "areas_total": 5,
  "areas_covered": 2,
  "coverage_pct": 40.0,
  "notes": "Areas derived from task.json goals + integration points: routing, data layer, auth, existing analog, tests.",
  "areas": {
    "routing": {
      "paths": ["src/routes/**/*.ts"],
      "priority": 1,
      "status": "covered",
      "files_total": 8,
      "files_inspected": 8,
      "lines_inspected": 612,
      "resume_marker": null,
      "log_path": ".claude/feature-state/log/research-routing.md",
      "integration_points": [
        {
          "file": "src/routes/documents.ts",
          "line_start": 41,
          "line_end": 58,
          "anchor": "app.get('/api/documents'",
          "note": "current list endpoint — search will be added as a query param here"
        }
      ],
      "reusable_primitives": [
        {
          "name": "withAuth",
          "file": "src/middleware/auth.ts",
          "line": 12,
          "anchor": "export function withAuth",
          "use": "every route uses this; search route must too"
        }
      ],
      "gaps": [
        { "id": "GAP-1", "description": "no shared query-parser helper", "needed_for": "AC-1 — parsing 'invoice' from query string into a search vector" }
      ],
      "assumption_verdicts": [
        { "assumption_id": "A-1", "verdict": "partial", "evidence": "src/db/migrations/2024-03-01.sql:14 — body is text, no tsvector yet" }
      ],
      "open_question_answers": []
    },
    "data-layer": {
      "paths": ["src/db/**", "src/models/**"],
      "priority": 2,
      "status": "partial",
      "files_total": 14,
      "files_inspected": 6,
      "lines_inspected": 380,
      "resume_marker": { "file": "src/models/document.ts", "line": 142, "reason": "token-limit" },
      "log_path": ".claude/feature-state/log/research-data-layer.md",
      "integration_points": [],
      "reusable_primitives": [],
      "gaps": [],
      "assumption_verdicts": [],
      "open_question_answers": []
    }
  }
}
```

**Valid `status` values** (lowercase): `not-started` · `partial` · `covered` · `research-failed`

**Valid `verdict` values for assumption_verdicts**: `confirmed` · `refuted` · `partial` · `unverifiable`

---

## `plan.json`

Written by `feature-planner`, reviewed by `plan-critic`, gated by user approval.

```json
{
  "built_at": "<iso8601>",
  "task_id": "add-server-side-search",
  "total_steps": 7,
  "notes": "First plan; no replans yet.",
  "steps_summary": [
    { "id": "STEP-0001", "order": 1, "title": "Add tsvector column to documents", "kind": "modify-file", "files": ["src/db/migrations/<new>.sql"], "depends_on": [] },
    { "id": "STEP-0002", "order": 2, "title": "Add SearchQuery type", "kind": "add-file", "files": ["src/types/search.ts"], "depends_on": [] }
  ],
  "deferred": [
    { "description": "Faceted UI for filters", "reason": "explicit non-goal in task.json" }
  ],
  "risks": [
    {
      "id": "RISK-1",
      "description": "tsvector index creation on a large table may lock writes",
      "likelihood": "medium",
      "impact": "high",
      "mitigation": "use CONCURRENTLY in the migration and verify in staging",
      "linked_steps": ["STEP-0001"]
    }
  ],
  "review": {
    "status": "draft",
    "critic_verdict": null,
    "critic_recommendations": [],
    "user_approved_at": null,
    "user_approval_text": null,
    "user_overrides": []
  }
}
```

**Valid `review.status` values**: `draft` · `approved` · `rejected`

**Valid `critic_verdict` values**: `confirm` · `reject` · `null` (not yet reviewed)

---

## `steps/STEP-NNNN.json`

All fields are required (use `null` for absent values, not omission). IDs are zero-padded four-digit integers (`STEP-0001`), assigned sequentially.

```json
{
  "id": "STEP-0003",
  "order": 3,
  "kind": "add-symbol",
  "title": "Add parseSearchQuery to src/search/parser.ts",
  "description": "Add a pure function `parseSearchQuery(input: string): SearchQuery` that produces the SearchQuery type from STEP-0002. No side effects. Mirrors the parseFilter pattern at src/search/parser.ts:14 (research note R-3).",
  "acceptance_criteria_covered": ["AC-1"],
  "depends_on": ["STEP-0002"],
  "files": [
    {
      "path": "src/search/parser.ts",
      "action": "modify",
      "anchor": "function parseFilter",
      "anchor_evidence": "src/search/parser.ts:14 (research-routing.md)",
      "intended_change": "Append `parseSearchQuery` immediately after `parseFilter`. Pure, no side effects, no validation (validation is STEP-0005)."
    }
  ],
  "interfaces": [
    "function parseSearchQuery(input: string): SearchQuery"
  ],
  "non_goals_for_step": [
    "do not modify parseFilter",
    "do not add validation here — that's STEP-0005",
    "do not export from a different module"
  ],
  "risks": [
    "if a later commit reorders helpers, the anchor may drift"
  ],
  "verification": {
    "static_checks": ["tsc --noEmit src/search/parser.ts", "eslint src/search/parser.ts"],
    "behavior_checks": [
      {
        "id": "BC-1",
        "description": "parseSearchQuery returns expected shape for plain input",
        "method": "unit-test",
        "expected": "{ terms: ['foo'], filters: [] }"
      }
    ],
    "regression_checks": ["existing parser tests still pass"],
    "anti_checks": ["diff confined to src/search/parser.ts within the anchor region"]
  },
  "requires_human_review": false,
  "breaking_change": false,
  "status": "planned",
  "linked_log": null,
  "linked_verifier_log": null,
  "implementer_dissent": null
}
```

**Valid `kind` values**: `add-file` · `modify-file` · `add-symbol` · `modify-symbol` · `add-test-fixture` · `scaffolding` · `wiring` · `refactor-prereq` · `doc-update`

**Valid `action` values for `files[]`**: `create` · `modify` · `delete`

**Valid `status` lifecycle**: `planned` → `in-progress` → `implemented` → `verified`
**Terminal:** `verified` · `wontfix` · `needs-human` · `deferred`

- `needs-human`: implementer or step-verifier raised dissent the user must resolve
- `deferred`: user chose to skip this step (recorded with reason)
- `wontfix`: rare — only for cases like the target file having been deleted between plan and implement

**Valid `method` values for `behavior_checks[]`**: `unit-test` · `integration-test` · `manual-repro` · `grep-anchor`

---

## `log/research-<area-id>.md`

Written by `project-researcher` per area. File path: `log/research-<area-id>.md`. Area id is kebab-case, never contains slashes.

Content: enumerated public surface, key flows with traces and anchors, conventions, reusable primitives, gaps, assumption verdicts, open question answers, integration points, STATUS marker. See `project-researcher` agent for the exact format.

---

## `log/step-STEP-NNNN.md`

Written by `feature-implementer`, then appended to by `test-engineer` and `step-verifier`.

Sections (in order of authorship):
1. **Step Record** — plan reference, pre-flight checks, change plan, edits applied, sanity checks, anti-checks, implementer observations (`feature-implementer`)
2. **Tests** — framework, tests written per BC, results across phases, `test_status` (`test-engineer`)
3. **Step-Verifier Verdict** — CONFIRM/REJECT with truth-by-truth evidence (`step-verifier`)

---

## `FINAL_REPORT.md`

Written by the orchestrator on `/feature:summary`. Only exists when the feature is complete (every step terminal). Do not write to this file manually — its contents are a factual reconstruction from the state files above.

---

## State transitions (summary)

```
Research area:  not-started → partial → covered
                                      └→ research-failed

Plan:           draft → approved
                     └→ rejected

Step:           planned → in-progress → implemented → verified
                                                    └→ needs-human
                                       └→ needs-human
                       └→ needs-human
                       └→ deferred (user skipped)
                                        wontfix (rare; e.g., file deleted)
```

---

## File-naming conventions

- Step ids: `STEP-NNNN`, zero-padded 4 digits, sequential.
- Research area logs: `log/research-<area-id>.md` where area-id is kebab-case (`auth`, `data-layer`, `existing-analog`).
- Step logs: `log/step-STEP-NNNN.md` matching the step id.
- Archived plans: `plan-<iso-date>.json` and `steps-<iso-date>/` directory.
- Archived tasks (when re-init'd): `archive/<iso-date>/task.json` etc.

---

## Cross-file invariants

- Every AC in `task.acceptance_criteria[]` must be referenced by at least one step's `acceptance_criteria_covered[]`. The plan-critic enforces this; the orchestrator double-checks at summary time.
- Every step's `files[*].anchor` must trace to either a `research.areas[*].integration_points[]` entry or a `gaps[]` entry. The plan-critic enforces this.
- Every research area in `research.json` must have a corresponding `log/research-<area-id>.md`. The orchestrator enforces this on each researcher return.
- Every step with `status: "implemented"` or later must have a `linked_log` pointing to an existing `log/step-STEP-NNNN.md`. Same for `linked_verifier_log` when status is `verified`.
- `plan.review.status == "approved"` is required before any step can transition `planned → in-progress`.
- `task.confirmed_by_user == true` is required before research starts.

---

## Why every state file is plain text

Chat transcripts don't survive `/clear`. JSON and Markdown do. Every agent reads from this ledger days later with full fidelity. State files are diffable, reviewable, and the only honest record of what happened.
