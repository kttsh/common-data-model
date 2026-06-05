# Core — Common Data Model API Platform

**Documentation-only repo. No code.** All tracked files are Markdown (25 files). Build/test/lint/package tooling does NOT exist — do not search for it.

Designs a "共通データモデル API 基盤": BigQuery → Dataform → FastAPI (Azure App Service) → Azure APIM → internal REST API consumers. Implementation code lives in *separate* repos (`dataform`, `api`); this repo is the design source-of-truth only.

## Invariants
- Body text = 日本語; file/folder names = English. User communication = 日本語.
- `CLAUDE.md` (repo root) = human-facing big picture: system diagram, owner table, known inconsistencies. Read it for the full architectural snapshot.
- `docs/README.md` = document index + "Compass" reading order.

## Folder roles (docs/)
- `01-requirements/` — goals, scope, constraints, open questions (baseline).
- `02-architecture/` — adopted decisions (architecture, runtime/FW, repo strategy).
- `03-authorization/` — authz design; `pycasbin/` holds impl memos.
- `04-research/` — raw research records. **Makes NO decisions** (decisions live in 02/03).
- `05-planning/` — roadmap, PoC scope.
- `06-data-platform/` — Dataform (naming, operating model, migration).

## Sub-memories
- Editorial rules (正本/owner principle) + durable design decisions: `mem:conventions`.
- Designed-system tech stack (Python/FastAPI/BigQuery/PyCasbin/Azure): `mem:tech_stack`.
- Available commands (and what's absent): `mem:suggested_commands`.
- What "done" means for a doc-editing task: `mem:task_completion`.