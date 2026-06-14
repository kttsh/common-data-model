# Runtime and Framework Decision (Python / FastAPI)

> 作成日: 2026-06-03

---

## サマリ

| 項目 | 決定 |
|---|---|
| 言語 | Python（基準 3.12+、当面の推奨 3.13。バージョンはコンテナイメージで固定） |
| フレームワーク | FastAPI（Starlette + Pydantic v2、ASGI、型ヒント駆動） |
| バージョン基準 | FastAPI 0.136 系（最新は 0.136.1 / 2026-04-23） |
| OpenAPI | 3.1（FastAPI のデフォルト出力）。APIM への取り込み時のみ 3.0 へダウングレードして投入 |
| 起動方式 | コンテナ CMD で `gunicorn -k uvicorn.workers.UvicornWorker`（ワーカー数・タイムアウトはイメージ/環境変数で制御） |
| 認証 | Entra ID JWT 検証（`fastapi-azure-auth` / `msal` / `PyJWT`） |
| 認可 | **PyCasbin（埋め込み PEP/PDP）で確定**、AuthZEN 準拠インターフェース（外部 PDP〔Cerbos/OPA〕は将来オプション） |
| デプロイ先 | Azure Red Hat OpenShift（ARO）上のコンテナ（Linux）、`api` リポジトリ。イメージは ACR に build-once で push し同一 digest を昇格 |

---

## 決定の前提

言語・FW の選定は、以下の確定済み前提を所与とする。

- **アーキテクチャ確定事項**: ARO 上の FastAPI コンテナ + 既存 APIM + APIM 組込みキャッシュ + プロセス内 LRU + BigQuery ビュー。
- **ランタイム制約 C1（2026-06 改定）**: バックエンドは ARO（Azure Red Hat OpenShift）上のコンテナ。メイン API は ARO 上の FastAPI コンテナ（旧制約「App Service / Functions のみ・コンテナ不可」は撤廃。正本は `platform-architecture-decision.md`）。
- **レイテンシ特性**: ホットパスは「Entra ID JWT 検証 → Open-GIM/SQL Server 属性参照 → BigQuery クエリ」で、待ち時間の支配項は外部 I/O。100〜500ms 目標もキャッシュヒット前提。
- **開発体制の第一指針**: Claude / Devin 等による AI 支援開発が機能しやすい、ドキュメント・コミュニティ・型情報が成熟したスタック。

この前提下では、フレームワーク単体の生スループットは決定打にならない。支配的な評価軸は、エコシステム成熟度・AI 支援開発との相性・Azure ネイティブ運用の容易性である。

---

## 選定理由

### エコシステム成熟度・実運用採用（リスク最小化）

- FastAPI は 2026 年時点で 1 日あたり約 450 万ダウンロード。OpenAI / Anthropic / Microsoft などが本番運用しており、Python の API FW としては最大規模のエコシステムを持つ。
- Q&A・Azure 公式チュートリアル・認証/認可/BigQuery 連携のサンプルが揃い、設計初期〜PoC でつまずきにくい。社内エンジニアの学習コストも最小。

### AI 支援開発との相性

- 学習データ量が最大の Python Web FW であり、Claude / Copilot / Cursor の補完精度が最も高い。
- Pydantic v2 は 2026 年の AI/LLM エコシステムの事実上の検証層であり、OpenAI SDK・Anthropic SDK・Google ADK・LangChain・LlamaIndex 等が採用する。Pydantic AI は「FastAPI の開発体験を AI エージェント開発に持ち込む」設計で、型・スキーマ資産がそのまま活きる。

### コンテナ運用との親和性（ARO）

- FastAPI / Uvicorn は ASGI 標準で、コンテナ化（Gunicorn + Uvicorn ワーカー）の定石が確立しており、ARO（OpenShift）上で素直に動く。イメージは ACR に build-once で push し、同一 digest を dev→stg→prod へ昇格する運用に乗せやすい。
- OpenShift の readiness / liveness probe・HPA・rolling / blue-green（Route 切替）/ canary をそのまま使える。PDP を別プロセスにしたい場合は同一 Pod のサイドカーとして同居でき、ネットワークホップを増やさずに済む（App Service では不可だった構成）。

### BigQuery / Entra ID / 認可ライブラリの成熟

- **BigQuery**: `google-cloud-bigquery`（同期）+ `google-cloud-bigquery-storage`（`BigQueryReadAsyncClient` で gRPC asyncio）。SDK は `HTTPS_PROXY` 環境変数を尊重するため、Proxy 経由アクセスはコード変更不要。
- **Entra ID JWT**: `fastapi-azure-auth`（FastAPI 特化・チュートリアル充実）、`fastapi-microsoft-identity`、`msal`（Microsoft 純正、OBO 対応）、`PyJWT` / `Authlib` が揃う。
- **認可**: 埋め込みの PyCasbin、外部 PDP の Cerbos / OPA（PlanResources / Compile API で行フィルタ AST を生成）まで選択肢が広い。**本基盤は PyCasbin（埋め込み）を採用**し、外部 PDP は将来オプションとして残す。AuthZEN 1.0 準拠の PEP を FastAPI で書く実装ガイドも 2026 年時点で整備済み。

### OpenAPI 3.1 スキーマファーストとの整合

FastAPI 0.136 系は OpenAPI 3.1 をデフォルト出力する。以前は 3.1 準拠が段階的という課題があったが 2026 年時点で解消済みで、スキーマファーストの設計方針と素直に合致する。

---

## 参考: モダンな代替スタック

本基盤は FastAPI を採用する。前提知識として、2026 年時点で実運用に耐えるモダンな API スタックを以下に挙げる（各スタックの 7 軸フラット比較・採用見送り理由は [`../04-research/api-runtime-framework-comparison-2026.md`](../04-research/api-runtime-framework-comparison-2026.md) が正本）。

- **Python / Litestar** — msgspec ベースで合成ベンチのスループットは最速級。型駆動・ASGI で FastAPI と設計思想は近い。
- **.NET 9 / Minimal API** — Azure・Entra ID・Managed Identity とのネイティブ統合が最も強い C# スタック。
- **Go / Huma** — OpenAPI 3.1 ファースト設計＋シングルバイナリの軽量デプロイ。
- **TypeScript / NestJS・Hono** — フロントと言語を揃えやすく、Hono はエッジ最適化に強い。
- **Azure Data API Builder（DAB）** — 設定駆動で REST/GraphQL を自動生成する Azure 系サービス。データソースが BigQuery のため対象外（DAB の対応ソースは Azure SQL / Cosmos DB / PostgreSQL / MySQL 等）として不採用。

---

## 参考リンク

- FastAPI Release Notes（0.136 系、OpenAPI 3.1 デフォルト）: https://fastapi.tiangolo.com/release-notes/
- fastapi · PyPI（最新版・Python 要件）: https://pypi.org/project/fastapi/
- Azure Red Hat OpenShift ドキュメント（マネージド OpenShift の概要・運用）: https://learn.microsoft.com/en-us/azure/openshift/
- Azure Red Hat OpenShift リリースノート（マネージド ID GA 等）: https://learn.microsoft.com/en-us/azure/openshift/azure-redhat-openshift-release-notes
- Network topology and connectivity for ARO（private cluster・egress lockdown・Route/Ingress）: https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/app-platform/azure-red-hat-openshift/network-topology-connectivity
- Litestar vs FastAPI（2026 分析、実運用では FastAPI 優位）: https://byteiota.com/litestar-vs-fastapi-python-speed-test-2026-analysis/
- Litestar vs FastAPI（Better Stack、エコシステム/採用トレードオフ）: https://betterstack.com/community/guides/scaling-python/litestar-vs-fastapi/
- Azure API Management — API formats support（OpenAPI 3.1 は import 互換のみ）: https://learn.microsoft.com/en-us/azure/api-management/api-management-api-import-restrictions
- Convert FastAPI OpenAPI 3.1 Specs to 3.0 for Azure APIM: https://medium.com/@rajeshroy362000/finally-convert-fastapi-openapi-3-1-specs-to-3-0-for-azure-apim-without-tears-b75d0bb85e57
- fastapi-azure-auth / fastapi-microsoft-identity（Entra ID JWT 検証）: https://pypi.org/project/fastapi-microsoft-identity/
- OpenShift Deployments（rolling / blue-green / canary 戦略）: https://docs.openshift.com/container-platform/latest/applications/deployments/deployment-strategies.html
- FastAPI Best Practices（ドメイン単位構成）: https://github.com/zhanymkanov/fastapi-best-practices

---

## 改訂履歴

| 日付 | revision | 内容 |
|---|---|---|
| 2026-06-04 | 0.1 | 案A 等の前提記述を整理し決定事項を直接記載。却下候補表を「参考: モダンな代替スタック」へ再構成 |
| 2026-06-04 | 0.2 | 他ドキュメントへのクロス参照（`*.md` §N 形式）を本文から除去 |
| 2026-06-04 | 0.3 | 節番号と本文中の自己参照を削除。重複・冗長表現を整理し、リンク集を「参考リンク」として分離 |
| 2026-06-11 | 0.4 | 代替スタックに Azure Data API Builder（不採用）を追記（旧 `data-api-builder-assessment.md` の決定を集約） |
| 2026-06-14 | 0.5 | **デプロイ先を App Service から ARO（コンテナ）へ変更**。C1 改定を反映し、起動方式をコンテナ CMD に、App Service 自動検出の根拠節を「コンテナ運用との親和性（ARO）」へ差し替え。参考リンクを ARO 系へ更新（FastAPI 採用判断自体は不変） |