# Implementation Roadmap

> Status: Working plan.
> Sources: `../01-requirements/product-requirements.md`, `../02-architecture/platform-architecture-decision.md`, and split research memos.

## Current Direction

- API platform mission: expose common data models built on BigQuery as internal REST APIs.
- Adopted platform baseline: App Service + existing APIM + APIM built-in cache + in-process LRU + BigQuery views.
- Authorization baseline: API/App Service owns per-user ABAC decisions; BigQuery executes pushed-down parameterized filters; APIM remains the coarse entry gate.
- Runtime/framework decision: **resolved — Python / FastAPI is adopted** (see `../02-architecture/runtime-framework-decision.md`). The earlier research shortlist (.NET Minimal API, Python Litestar/FastAPI, Go/Huma) is retained only as re-evaluation triggers.

## Immediate Design Work

1. Confirm Proxy specifications and run an App Service to BigQuery connectivity PoC.
2. Confirm App Service plan constraints, Linux/Windows constraints, and deployment slot availability.
3. Confirm existing APIM Product/Subscription naming and tenancy rules.
4. Confirm Open-GIM / on-prem SQL Server schema, join key from Entra ID, and available authorization attributes.
5. ~~Finalize runtime/framework choice~~ **(done: Python / FastAPI)** and define the OpenAPI workflow (FastAPI 3.1 output -> downgrade to 3.0 for APIM import).
6. Design the authorization policy format and AST-to-BigQuery translation layer.
7. Define audit log schema, especially subject/action/resource, decision reason, and applied filter.

## PoC Scope

1. `/healthz` endpoint and OpenAPI publication.
2. APIM -> App Service -> BigQuery via Proxy latency measurement.
3. Entra ID JWT verification -> Open-GIM attribute lookup -> authorization decision.
4. Row-level filter generation and parameterized BigQuery query execution.
5. Column mask application and audit logging.
6. APIM built-in cache and process-local LRU cache behavior measurement.

## Research Backlog

- `../02-architecture/runtime-framework-decision.md`: **runtime/framework decision (Python / FastAPI, resolved 2026-06)**.
- `../04-research/api-runtime-framework-comparison-2026.md`: runtime/framework comparison (research input behind the decision above).
- `../04-research/ci-cd-delivery-research-2026.md`: CI/CD・SBOM・STO・SRM 調査（**Harness 一式は採用見送り＝調査資料**。実装は **GitHub Actions** を採用）。
- `../04-research/authorization-models-and-standards-2026.md`: ABAC/PBAC/AuthZEN/RFC background.
- `../02-architecture/repository-structure-options.md`: source tree options by runtime.

## Caveats From Current Research

### Recommendations

1. **Phase 0(チーム前提整理・1〜2週間)**
   - 言語・FW は **Python / FastAPI に確定済み**(`../02-architecture/runtime-framework-decision.md`)。スキルセット棚卸しは「Python/FastAPI への習熟度・OpenAPI スキーマファースト運用」に観点を移す。.NET 経験者が大半など分布が大きく偏った場合のみ再検討トリガーとして扱う。
   - Bicep / Terraform の社内方針確認 → IaC 方針(Terraform 推奨)を最初に決める(後戻り高コスト)。
2. **Phase 1(PoC・4〜6週間)**
   - 選定言語で最小エンドポイント(`/healthz` + BigQuery 簡易クエリ + Entra ID JWT 検証)を実装、App Service Linux にデプロイ。
   - GitHub Actions の CI/CD 雛形(.github/workflows/ci.yml + deploy-dev.yml)を1本動かす(build → ACR push → App Service デプロイスロット swap)。Harness は採用見送り(`../04-research/ci-cd-delivery-research-2026.md`、将来比較用)。
   - PyCasbin(埋め込み)で ABAC を1ルールだけ通す(`../03-authorization/pycasbin/`)。
3. **Phase 2(本格構築・8〜12週間)**
   - STO(Trivy + Snyk + Spectral + Schemathesis)を CI に組込み、SBOM を ACR に署名付きで push。
   - SRM の Azure Log Analytics ヘルスソース連携、SLO 99.9% + エラーバジェット連動 Canary ロールバック。
   - APIM Policy で JWT 事前検証(発行者・audience)+ App Service で再検証(claims ベース ABAC 用)。
4. **しきい値(判断を変える基準)**
   - LLM 補完が業務時間の30%未満しか節約しない → AI 相性軸の優先度を下げ、.NET 一択でよい。
   - チームが10名超 → 構造強制が必要なため Litestar より FastAPI、Fastify より NestJS。
   - コールドスタートが SLO に影響 → Quarkus Native か .NET Native AOT への切替検討。

---

### Caveats

1. **ベンチマーク数値の出典・条件**: Encore.ts「Express 比 9倍、Hono・Elysia 比 3倍」は Encore 公式エンジニアリングブログ(encore.dev/blog/event-loops)、Fastify「Express 比 ~4.95倍」は Fastify 公式ベンチ(fastify.dev/benchmarks/、2026年1月1日更新)、Litestar「Pydantic v2 比 ~12倍速」は jcrist/msgspec 公式ベンチ(JSON decode + validate、CPython 3.11、x86 Linux)。社内ワークロード(BigQuery + SQL Server の I/O 待機が支配的)では差が縮まる可能性大。
2. **Microsoft 公式 .NET 9 改善幅**: 発表ブログ(devblogs.microsoft.com/dotnet/announcing-dotnet-9/)は「higher throughput and a dramatic drop in memory usage」「The memory drop is due to the Server GC changes」と定性的記述のみ。コミュニティブログで広まる「+15% / -93%」の具体数値は Microsoft 公式で直接確認できないため留保。
3. **Harness ステップの YAML `type:` 文字列**: Azure Web App 系は公式ドキュメントが UI 名(Slot Deployment / Traffic Shift など)で記述しており、YAML schema 文字列(`AzureSlotDeployment` 等)は Harness の Visual→YAML 切替か `github.com/harness/harness-schema/blob/main/v0/pipeline.json` で最終確認推奨。
4. **Bicep / IaCM**: Harness IaCM は Terraform / OpenTofu のみ GA、Pulumi / Crossplane は roadmap(2026年5月時点、TechTarget が Harness PM の Rohit Reddy Kaliki のコメントを掲載)、Bicep は ARM の JSON に変換しない限り IaCM 経由では扱えない。
5. **Feature Flags の Split.io 統合**: Harness は Split.io 買収後 FME(Feature Management & Experimentation)へリブランド中、UI / SDK は移行期。Feature Flags の歴史的初期価格 $15/dev/月(TechTarget、2021年)は現行価格と異なる可能性あり、`harness.io/pricing` で確認すること。
6. **Bun / Elysia / Encore.ts**: Azure App Service Linux の公式サポート対象外であり、今回の制約下では実用面で除外したが、Container Apps 解禁時には再評価の余地あり。
7. **AI 駆動開発との相性**: 学習データ量に依拠する評価で、LLM のモデル更新(Claude 4 系、GPT-5 系など)で順位が変動する可能性が高い軸であることに留意。
