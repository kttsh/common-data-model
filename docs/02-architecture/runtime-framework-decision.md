# Runtime and Framework Decision (Python / FastAPI)

> 作成日: 2026-06-03

---

## サマリ

| 項目 | 決定 |
|---|---|
| 言語 | Python（基準 3.12+、当面の推奨 3.13。App Service の FastAPI 自動検出は 3.14+） |
| フレームワーク | FastAPI（Starlette + Pydantic v2、ASGI、型ヒント駆動） |
| バージョン基準 | FastAPI 0.136 系（最新は 0.136.1 / 2026-04-23） |
| OpenAPI | 3.1（FastAPI のデフォルト出力）。APIM への取り込み時のみ 3.0 へダウングレードして投入 |
| 起動方式 | `gunicorn -k uvicorn.workers.UvicornWorker`（FastAPI 自動検出が効く環境では起動コマンドを省略可） |
| 認証 | Entra ID JWT 検証（`fastapi-azure-auth` / `msal` / `PyJWT`） |
| 認可 | **PyCasbin（埋め込み PEP/PDP）で確定**、AuthZEN 準拠インターフェース（外部 PDP〔Cerbos/OPA〕は将来オプション） |
| デプロイ先 | Azure App Service（Linux）、`api` リポジトリ |

---

## 決定の前提

言語・FW の選定は、以下の確定済み前提を所与とする。

- **アーキテクチャ確定事項**: App Service + 既存 APIM + APIM 組込みキャッシュ + プロセス内 LRU + BigQuery ビュー。
- **ランタイム制約 C1**: バックエンドは Azure App Service / Functions のみ（Container Apps / AKS 不可）。メイン API は App Service。
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

### Azure App Service ネイティブ対応の強化（2026）

- 2026-05-18 の Azure App Service チームの発表により、App Service for Linux が FastAPI を自動検出するようになった（`main.py` / `app.py` / `asgi.py` 等を走査して FastAPI import を検出）。検出時は Gunicorn + Uvicorn ワーカークラスで自動起動し、従来必要だった起動コマンドの手動設定が不要になる。
- 同発表で Python のデプロイ遅延が約 30% 短縮。本機能は当面 Python 3.14 以降で有効（以降のバージョンへ順次拡大予定）。

### BigQuery / Entra ID / 認可ライブラリの成熟

- **BigQuery**: `google-cloud-bigquery`（同期）+ `google-cloud-bigquery-storage`（`BigQueryReadAsyncClient` で gRPC asyncio）。SDK は `HTTPS_PROXY` 環境変数を尊重するため、Proxy 経由アクセスはコード変更不要。
- **Entra ID JWT**: `fastapi-azure-auth`（FastAPI 特化・チュートリアル充実）、`fastapi-microsoft-identity`、`msal`（Microsoft 純正、OBO 対応）、`PyJWT` / `Authlib` が揃う。
- **認可**: 埋め込みの PyCasbin、外部 PDP の Cerbos / OPA（PlanResources / Compile API で行フィルタ AST を生成）まで選択肢が広い。**本基盤は PyCasbin（埋め込み）を採用**し、外部 PDP は将来オプションとして残す。AuthZEN 1.0 準拠の PEP を FastAPI で書く実装ガイドも 2026 年時点で整備済み。

### OpenAPI 3.1 スキーマファーストとの整合

FastAPI 0.136 系は OpenAPI 3.1 をデフォルト出力する。以前は 3.1 準拠が段階的という課題があったが 2026 年時点で解消済みで、スキーマファーストの設計方針と素直に合致する。

---

## 参考: モダンな代替スタック

本基盤は FastAPI を採用する。前提知識として、2026 年時点で実運用に耐えるモダンな API スタックを以下に挙げる。

- **Python / Litestar** — msgspec ベースで合成ベンチのスループットは最速級。型駆動・ASGI で FastAPI と設計思想は近い。
- **.NET 9 / Minimal API** — Azure・Entra ID・Managed Identity とのネイティブ統合が最も強い C# スタック。
- **Go / Huma** — OpenAPI 3.1 ファースト設計＋シングルバイナリの軽量デプロイ。
- **TypeScript / NestJS・Hono** — フロントと言語を揃えやすく、Hono はエッジ最適化に強い。

---

## 参考リンク

- FastAPI Release Notes（0.136 系、OpenAPI 3.1 デフォルト）: https://fastapi.tiangolo.com/release-notes/
- fastapi · PyPI（最新版・Python 要件）: https://pypi.org/project/fastapi/
- Simplifying FastAPI Deployments on Azure App Service for Linux（FastAPI 自動検出・Gunicorn+Uvicorn 自動構成、2026-05）: https://techcommunity.microsoft.com/blog/appsonazureblog/simplifying-fastapi-deployments-on-azure-app-service-for-linux/4520103
- Platform Improvements for Python AI Apps on Azure App Service（デプロイ遅延 ~30% 改善、2026-05-18）: https://azure.github.io/AppService/2026/05/18/platform-improvements-for-python-ai-apps-on-azure-app-service.html
- Python 3.14 on Azure App Service for Linux: https://azure.github.io/AppService/2025/10/28/python314-available.html
- Litestar vs FastAPI（2026 分析、実運用では FastAPI 優位）: https://byteiota.com/litestar-vs-fastapi-python-speed-test-2026-analysis/
- Litestar vs FastAPI（Better Stack、エコシステム/採用トレードオフ）: https://betterstack.com/community/guides/scaling-python/litestar-vs-fastapi/
- Azure API Management — API formats support（OpenAPI 3.1 は import 互換のみ）: https://learn.microsoft.com/en-us/azure/api-management/api-management-api-import-restrictions
- Convert FastAPI OpenAPI 3.1 Specs to 3.0 for Azure APIM: https://medium.com/@rajeshroy362000/finally-convert-fastapi-openapi-3-1-specs-to-3-0-for-azure-apim-without-tears-b75d0bb85e57
- fastapi-azure-auth / fastapi-microsoft-identity（Entra ID JWT 検証）: https://pypi.org/project/fastapi-microsoft-identity/
- Tutorial: Deploy a Python FastAPI Web App（Azure 公式）: https://learn.microsoft.com/en-us/azure/app-service/tutorial-python-postgresql-app-fastapi
- FastAPI Best Practices（ドメイン単位構成）: https://github.com/zhanymkanov/fastapi-best-practices

---

## 改訂履歴

| 日付 | revision | 内容 |
|---|---|---|
| 2026-06-04 | 0.1 | 案A 等の前提記述を整理し決定事項を直接記載。却下候補表を「参考: モダンな代替スタック」へ再構成 |
| 2026-06-04 | 0.2 | 他ドキュメントへのクロス参照（`*.md` §N 形式）を本文から除去 |
| 2026-06-04 | 0.3 | 節番号と本文中の自己参照を削除。重複・冗長表現を整理し、リンク集を「参考リンク」として分離 |