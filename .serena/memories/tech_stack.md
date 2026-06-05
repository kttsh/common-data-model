# Tech Stack

**This repo itself has no build stack** (Markdown docs only). Below is the *designed system's* confirmed stack — durable design facts, not code present here.

- **Language/FW**: Python (3.12+ baseline, 3.13 recommended; App Service FastAPI auto-detect needs 3.14+) / FastAPI 0.136 series / Starlette + Pydantic v2 / ASGI.
- **API spec**: OpenAPI 3.1 (FastAPI default). Downgrade to 3.0 only for APIM import.
- **Runtime host**: Azure App Service (Linux). Constraint C1: only App Service / Functions (no Container Apps / AKS). Start cmd `gunicorn -k uvicorn.workers.UvicornWorker`.
- **Gateway**: existing shared Azure APIM (front: JWT pre-check, throttle, subscription, built-in cache).
- **Data**: BigQuery via Proxy. `google-cloud-bigquery` (+ `-storage` async). SDK honors `HTTPS_PROXY` → no code change for proxy.
- **Authz**: PyCasbin embedded (PERM model.conf + CSV policies). Cerbos/OPA = future option.
- **AuthN**: Entra ID JWT (`fastapi-azure-auth` / `msal` / `PyJWT`); SA via Client Credentials / Managed Identity; OBO = RFC 8693.
- **Attribute store**: Open-GIM (reached via ExpressRoute).
- **Cache**: APIM built-in + in-process LRU only. Redis NOT approved (constraint C2).
- **Data modeling**: Dataform (SQLX, workflow_settings.yaml) → BigQuery, deployed via @dataform/cli.
- **CI/CD**: GitHub Actions (Harness rejected). OIDC auth (GCP=Workload Identity Federation, Azure=federated credentials).
- **Persistence (if ever)**: Azure SQL only approved (constraint C3 — no Cosmos/Postgres).
- **Audit/observability**: Azure Log Analytics (Datadog optional).