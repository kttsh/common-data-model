# Runtime and Framework Decision (Python / FastAPI)

> 作成日: 2026-06-03
> ステータス: **Python / FastAPI 採用決定**
> 情報時点: 2026年6月（本文の数値・対応状況は 2026 年の一次情報に基づく）
> Related: `platform-architecture-decision.md`（D1 をクローズ）, `../01-requirements/product-requirements.md`（§8 / L4）, `../04-research/api-runtime-framework-comparison-2026.md`, `repository-structure-options.md`, `data-api-builder-assessment.md`

---

## 1. 決定事項サマリ

| 項目 | 決定 |
|---|---|
| **言語** | **Python**（3.12+ を基準、3.13 を当面の推奨。App Service の FastAPI ネイティブ検出は 3.14+） |
| **フレームワーク** | **FastAPI**（Starlette + Pydantic v2、ASGI、型ヒント駆動） |
| バージョン基準 | FastAPI **0.136 系**（2026-04-23 リリースの 0.136.1 が当時点の最新） |
| OpenAPI | **3.1**（FastAPI のデフォルト出力）。ただし APIM 取り込み時は **3.0 へダウングロード**して投入（§5・R1） |
| 起動方式 | `gunicorn -k uvicorn.workers.UvicornWorker`（App Service Linux の FastAPI 自動検出が効く環境では起動コマンド省略可、§3.4） |
| 認証 | Entra ID JWT 検証（`fastapi-azure-auth` / `msal` / `PyJWT`） |
| 認可 | 埋め込み PEP/PDP（PyCasbin または自前 + CEL）から開始、AuthZEN 準拠インターフェース（`../03-authorization/authorization-boundaries-and-interface.md`） |
| デプロイ先 | Azure App Service（Linux）。`../06-data-platform/repository-strategy.md` の `api` リポジトリ |

> **この決定は `platform-architecture-decision.md` §8.2 の判断項目 D1（言語・FW 選定）をクローズする。** 同時に `../01-requirements/product-requirements.md` の論点 L4（API バックエンドのランタイム）のうち、ランタイム＝App Service は既決、言語・FW＝FastAPI を本書で確定する。

---

## 2. 決定の背景とスコープ

本決定は、以下の確定済み前提の上で言語・FW を選ぶものである。

- **アーキテクチャは案A（最小構成）確定**：App Service + 既存 APIM + APIM 組込みキャッシュ + プロセス内 LRU + BigQuery ビュー（`platform-architecture-decision.md`）。
- **ランタイム制約 C1**：バックエンドは Azure App Service / Functions のみ（Container Apps / AKS 不可）。メイン API は App Service。
- **レイテンシ特性**：ホットパスは「Entra ID JWT 検証 → OpenGIM/SQL Server 属性参照 → BigQuery クエリ」で、**待ち時間の支配項は外部 I/O**。100〜500ms 目標もキャッシュヒット前提（`product-requirements.md` §6）。
- **開発体制の第一指針**：Claude / Devin 等による **AI 支援開発が機能しやすい、ドキュメント・コミュニティ・型情報が成熟したスタック**（`product-requirements.md` §8）。

この前提下では「フレームワーク単体の生スループット」は決定的な評価軸にならず、**エコシステム成熟度・AI 支援開発相性・Azure ネイティブ運用容易性**が支配的な評価軸になる。

---

## 3. 選定理由（肉付け）

### 3.1 本構成では「FW の生スループット」が KPI にならない

`api-runtime-framework-comparison-2026.md` の Python 総合所見は「**Litestar が技術的にはベストだが、エコシステム成熟度・採用事例・社内学習コストを考慮すると FastAPI が現実解**」だった。本構成は、この所見を裏づける条件がすべて揃っている。

- Litestar の優位の核は msgspec による JSON デコード+バリデーションの高速化（jcrist/msgspec 公式ベンチで Pydantic v2 比 ~12倍）だが、**本基盤の応答時間は BigQuery・SQL Server の I/O 待機が支配項**であり、シリアライズ速度の差は体感レイテンシにほぼ現れない。
- 2026 年の比較記事も「**Litestar は合成ベンチで勝つが、FastAPI がほぼすべての実運用判断で勝つ。その差はユーザー体感レイテンシにめったに現れない**（Litestar wins synthetic charts; FastAPI wins almost every real production decision）」と総括しており、本件の I/O 支配ワークロードと整合する。
- したがって、純性能を理由に Litestar を採る積極的根拠は本構成では希薄。`implementation-roadmap.md` の「しきい値」にある『チームが10名超なら構造強制のため Litestar より FastAPI』という判断とも矛盾しない。

### 3.2 エコシステム成熟度・実運用採用（リスク最小化）

- FastAPI は **2026 年時点で 1 日あたり約 450 万ダウンロード**、OpenAI / Anthropic / Microsoft などが本番運用しており、Python の API FW としては最大規模のエコシステム。
- Stack Overflow の Q&A、Azure 公式チュートリアル、認証・認可・BigQuery 連携のサンプルが揃っており、設計初期〜PoC で詰まりにくい。社内エンジニアの学習コストも最小。

### 3.3 AI 支援開発との相性（要件 §8 の第一指針）

- 学習データ量が最大の Python Web FW であり、Claude / Copilot / Cursor の補完精度が最も高い（`api-runtime-framework-comparison-2026.md` §1.1）。「FastAPI と勘違いした出力が混じる」Litestar のような取り違えリスクが小さい。
- **Pydantic v2 は 2026 年の AI/LLM エコシステムの事実上の検証層**：OpenAI SDK・Anthropic SDK・Google ADK・LangChain・LlamaIndex 等が Pydantic Validation を採用。Pydantic AI は「FastAPI の開発体験を AI エージェント開発に持ち込む」設計で、型・スキーマ資産がそのまま活きる。
- これは将来の **AI エージェント連携（OBO/委譲で人の所属に絞る、`../03-authorization/authorization-models-and-standards-2026.md` §3.1）** と地続きで、スタックを薄く保てる。

### 3.4 Azure App Service ネイティブ対応の 2026 年強化（C1 制約に最適）

- **2026-05-18 の Azure App Service チームの発表**で、App Service for Linux が **FastAPI を自動検出**するようになった（`main.py` / `app.py` / `asgi.py` 等のエントリを走査し FastAPI import を検出）。検出時は **Gunicorn + Uvicorn ワーカークラスで自動起動**し、従来必要だった起動コマンドの手動設定が不要になる。
- 同発表で **Python デプロイ遅延が約 30% 短縮**。本機能は当面 **Python 3.14 以降**で有効（以降のバージョンへ順次拡大予定）。
- コンテナ不可（C1）の本構成では、**コード方式デプロイがコンテナと同等に素直**になる点が大きく、App Service との適合度が高い。

### 3.5 BigQuery / Entra ID / 認可 ライブラリの成熟

- **BigQuery**：`google-cloud-bigquery`（同期）+ `google-cloud-bigquery-storage`（`BigQueryReadAsyncClient` で gRPC asyncio）。SDK は `HTTPS_PROXY` 環境変数を尊重するため Proxy 経由アクセスはコード変更不要（`repository-structure-options.md` §3.2）。
- **Entra ID JWT**：`fastapi-azure-auth`（FastAPI 特化・チュートリアル充実）、`fastapi-microsoft-identity`、`msal`（Microsoft 純正、OBO 対応）、`PyJWT`/`Authlib` が揃う。
- **認可**：埋め込みの PyCasbin、外部 PDP の Cerbos / OPA（PlanResources / Compile API で行フィルタ AST 生成）まで選択肢が広く、AuthZEN 1.0 準拠の PEP を FastAPI で書く実装ガイドも 2026 年時点で整備済み（`../03-authorization/authorization-models-and-standards-2026.md` §4）。

### 3.6 OpenAPI 3.1 スキーマファーストとの整合

- FastAPI 0.136 系は **OpenAPI 3.1 をデフォルト出力**する。`api-runtime-framework-comparison-2026.md` が記した「OpenAPI 3.1 完全準拠は依然として段階的」という評価は **2026 年時点では解消**しており、要件 §5 / 設計確定事項 L5・L6（OpenAPI 3.1 準拠・スキーマファースト）と素直に噛み合う。

---

## 4. 却下した候補と理由

| 候補 | 却下理由（本構成において） |
|---|---|
| **Litestar** | 技術的には魅力だが、優位の核（msgspec シリアライズ高速化）が I/O 支配の本構成で体感に出ない。エコシステム・採用事例・AI 補完の学習データ・社内学習コストで FastAPI に劣後。大規模化が顕在化したら再検討（§6）。 |
| **.NET 9 Minimal API** | Azure ネイティブ統合・Entra ID/Managed Identity 純正は最強だが、**AI 支援開発相性（要件 §8 の第一指針）とデータ/ML エコシステム親和性で Python に劣る**。BigQuery クライアントの成熟度・サンプル量も Python が上。チームの .NET 比率が高い場合のみ再考。 |
| **Go / Huma** | OpenAPI 3.1 ファースト設計と AI 補完精度は高いが、**コミュニティ規模が小さく商用サポートなし**。BigQuery + 認可 + Entra のサンプル充実度で Python に劣る。シングルバイナリの軽量性は C1 制約下では決定打にならない。 |
| **TypeScript（NestJS / Hono ほか）** | 要件 §8 の候補だが、データ/ML 親和性と BigQuery・認可エコシステムで Python に対する優位がない。Hono のエッジ最適化は App Service 前提の本構成で旨味が出ない（`api-runtime-framework-comparison-2026.md` §1.2）。 |

---

## 5. 既知の注意点・リスクと緩和策

| # | 注意点 | 緩和策 |
|---|---|---|
| R1 | **既存 APIM は OpenAPI 3.1 を完全サポートしない**。2026 年時点でも APIM は 3.1 を「import 互換のみ・feature 非互換」として扱い、3.1 固有構文の一部は無視/3.0 相当へ降格される | FastAPI が出力する 3.1 スキーマを **3.0 へダウングロードしてから APIM へ投入**するステップを CI に組み込む（nullable 型・`const`→単値 enum・条件スキーマ→`oneOf` 変換等）。スキーマ正本はリポジトリ側に保持し diff 比較 |
| R2 | **Pydantic 起因のシリアライズ性能頭打ち**（大量行・大ペイロード時） | 行ストリームは BigQuery Storage Read API（async）で取得。ページング既定（`pagination.py`）。それでも頭打ちなら該当エンドポイントのみ msgspec/orjson レスポンス化を検討 |
| R3 | **大規模化時のルーター循環 import / 構成の崩れ** | 起点から **ドメイン別ディレクトリ構成**（FastAPI Best Practices / Netflix Dispatch 由来）を採用。各ドメインが router/schemas/service/repository を自己完結で持つ（`../06-data-platform/repository-strategy.md` §4 準拠） |
| R4 | **App Service の FastAPI ネイティブ検出は Python 3.14+ のみ** | 3.14 未満を使う期間は従来どおり起動コマンド（`gunicorn -k uvicorn.workers.UvicornWorker`）を明示。3.14+ へ移行後に省略可 |
| R5 | **CPU バウンド処理での GIL 制約** | 本構成は I/O バウンドが支配的なため影響小。重い整形・集計は BigQuery 側へ押し下げ、worker 数（プロセス）で水平に捌く |

---

## 6. 再検討トリガー

| トリガー | 候補となる見直し |
|---|---|
| AI 補完が業務時間の 30% 未満しか節約しない（`implementation-roadmap.md` しきい値） | AI 相性軸の優先度を下げ、Azure ネイティブ統合最強の .NET 9 Minimal API を再評価 |
| ワークロードが CPU バウンド寄りに変化 / 生スループットが SLO を律速 | 該当サービスのみ Litestar・Go/Huma へ部分移行を検討 |
| Container Apps / AKS が解禁（`platform-architecture-decision.md` §7） | ランタイム全体の再評価（OPA サイドカー構成等が選択肢に） |
| チームの言語スキル分布が大きく偏った（.NET / Go 経験者中心） | Phase 0 のスキルセット棚卸し結果に応じて再判断 |

---

## 7. 参考（2026 年出典）

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
- Pydantic AI（FastAPI 流の AI エージェント開発、Pydantic が各社 SDK の検証層）: https://github.com/pydantic/pydantic-ai
- Tutorial: Deploy a Python FastAPI Web App（Azure 公式）: https://learn.microsoft.com/en-us/azure/app-service/tutorial-python-postgresql-app-fastapi
- FastAPI Best Practices（ドメイン単位構成）: https://github.com/zhanymkanov/fastapi-best-practices

---

## 8. 改訂履歴

| 日付 | 版 | 内容 |
|---|---|---|
| 2026-06-03 | 1.0 | Python / FastAPI 採用決定。D1（言語・FW 選定）をクローズ。2026 年の一次情報で選定理由を肉付け |
