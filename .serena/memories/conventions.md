# Conventions — editorial rules & durable design decisions

## 正本 (owner) principle — THE core editing rule
Each topic has exactly ONE owner doc. Other docs carry only summary + relative-path link. **Never duplicate the substance.** Before writing a fact/decision, find its owner (below) and write the body only there.

Owner table:
- Terms/abbreviations → `docs/03-authorization/glossary.md` (no per-doc abbrev tables; add new terms only here).
- Platform architecture decision (案A) → `docs/02-architecture/platform-architecture-decision.md`.
- Language/FW decision (Python/FastAPI) → `docs/02-architecture/runtime-framework-decision.md`.
- FW comparison (raw) → `docs/04-research/api-runtime-framework-comparison-2026.md`.
- Authz library comparison + PyCasbin rationale → `docs/04-research/abac-authz-library-comparison.md`.
- PEP/PDP responsibilities + `authorize()` interface → `docs/03-authorization/authorization-boundaries-and-interface.md`.
- Repo split / branching / promotion → `docs/02-architecture/repository-strategy.md`.
- Dataform structure/naming → `docs/06-data-platform/dataform-naming-convention.md`.
- Dataform migration (CLI route) → `docs/06-data-platform/migration-plan.md`.

Rules: 04-research never records decisions. Cross-references use relative paths (`../02-architecture/...`); consolidate into link sections, don't scatter inline. Decision docs carry 改訂履歴 tables — update when changing a decision.

## Durable design decisions (settled)
- Architecture = **案A** (minimal: zero added Azure services). 案B/案C kept for future comparison only.
- Authz = App Service ABAC, **2-stage**: ① coarse gate (allow/deny, deny→403 short-circuit) → ② generate `row_filter` (AST) + `masked_columns`. FastAPI translates AST → parameterized WHERE, pushes down to BigQuery. BigQuery RLS NOT used (avoid double management).
- Engine = **PyCasbin embedded PEP/PDP** (confirmed). Cerbos/OPA = future option. PEP↔PDP designed AuthZEN 1.0-aware for swappability.
- Authz attribute source-of-truth = **Open-GIM** (whether same entity as on-prem SQL Server is an open question).
- Repo split = **案B** (deploy boundary = repo boundary). `dataform` and `api` separate. Entities (Order/Invoice/…) grow as per-entity directories, not new repos = modular monolith. 1 entity ↔ URL ↔ `src/<entity>/` ↔ `definitions/outputs/<entity>/` ↔ `policies/<entity>.csv`, same name 1:1.
- CI/CD = **GitHub Actions** (Harness rejected). build-once/promote (same digest/commit dev→stg→prod), fix-forward (never edit envs directly), trunk-based (protected main + squash + approval gates).
- Dataform deploy = **@dataform/cli** (env diff via `--default-database` etc.). GCP-native operation rejected.

## Known inconsistency
`docs/README.md` links to `02-architecture/data-api-builder-assessment.md` and `02-architecture/repository-structure-options.md` — both MISSING. Watch for such dangling links when editing.