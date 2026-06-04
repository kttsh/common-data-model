# Data API Builder Assessment

> Source: split from authorization strategy memo.
> Status: Design decision input.

## Summary

### 2.1 まず一言で（技術に明るくない人向け）

DAB（Data API Builder）は「データベースを置くだけで、コードを書かずに自動で REST / GraphQL API ができる」便利ツール。設定ファイル1枚で独自 API を書かずにデータベースを公開でき、一般的な CRUD 操作の独自 API を置き換えるもの。

便利な反面、「自動で作れる」代わりに「決まった型からはみ出しにくい」。今回やりたいことの本質は、まさにその “はみ出す” 部分（BigQuery を相手にする／属性を社外リポジトリから引いて認可する／統合ロジックを組む）にある。だから DAB の強みは効かず、弱みだけが残る ── これが不採用の理由。

### 2.2 なぜ合わないか（技術的な決定打）

**1. DAB は BigQuery に対応していない（これだけで採用不可）**
DAB の対応データソースは SQL Server / Azure SQL、Azure Cosmos DB、PostgreSQL、MySQL のみで、BigQuery は含まれない。本基盤の一次データソースは BigQuery なので、DAB はそもそも接続できない。理論武装としてはこの一点で十分。

**2. DAB の認可は「トークンのクレーム」と「行のフィールド」しか見られない**
DAB の行レベル制御はデータベースポリシー（OData スタイルの述語が WHERE 句に変換される）で実装し、式の中で参照できるのは `@item.<フィールド>`（行の項目）と `@claims.<クレーム>`（トークンのクレーム）。つまり「認可に必要な属性が JWT に入っている」ことが大前提。今回は役職・所属が Entra ID トークンに入っておらず、別の社内リポジトリを実行時に引く必要がある（Part 1 ①）。DAB にはリクエスト処理の途中で外部の属性ストアを引いてポリシーに合流させる仕組みがない。要件と認可モデルが構造的に噛み合わない。

**3. DAB は「アプリのコードを書かない」思想なので、ロジック層を置けない**
DAB の売りは設定ファイルだけでコード不要という点で、汎用的な CRUD 操作の独自 API を置き換えるもの。今回中心になるのは、未払＋計上のユニオン＋重複排除、ユーザー文脈に応じたモデル組み立て、列マスク、適用フィルタの監査ログ ── いずれもコードでしか書けない処理。DAB は CRUD 自動化の置き換えであって、ビジネスロジック層の置き換えではない。

**4. ロール選択がクライアント申告（ヘッダ）依存**
DAB は 1 リクエストにつき 1 つの有効ロールで評価し、Anonymous / Authenticated 以外のロールを使うにはクライアントが `X-MS-API-ROLE` ヘッダを送る必要がある。「クライアントが自分のロールを宣言する」モデルは、サーバー側で属性から動的に範囲を決める今回の方針と相性が悪く、信頼境界の観点でも扱いづらい。

**【公平に】DAB が正解になる条件**（「なぜ検討したのか」と問われた時の保険）
データが Azure SQL 等の対応 DB にあり、かつ認可属性がトークンのクレームに乗っている単純な CRUD 公開なら、DAB は最速の正解。今回はその両方を満たさないから外す、という整理が最もフェアで強い。

### 2.3 では FastAPI で何が嬉しいか

FastAPI なら、JWT 検証・社内リポジトリからの属性取得・BigQuery クエリ・ABAC 判定・列マスク・監査ログを、すべて自前のコードで 1 つのアプリ内に組み合わせられる。

- BigQuery クライアントは `HTTPS_PROXY` を尊重するので閉域 Proxy 経由でもコード変更不要。
- Entra ID 検証は `msal` / `PyJWT`。
- ポリシーは PyCasbin（採用・埋め込み）等と組める。
- OpenAPI 3.1 ネイティブ、AI 支援開発との相性も良好。

### 2.4 App Service か Functions か（2026 ベストプラクティス）

| 観点 | Azure App Service | Azure Functions |
|---|---|---|
| 形態 | 永続プロセス（PaaS） | イベント駆動・サーバーレス |
| コールドスタート | 常時稼働プランでは無し | 従量ではゼロスケール → 発生 |
| フル REST API 適性 | ◎ フレームワーク／ルーティング／ミドルウェアが素直 | △ コントローラ非対応で full REST はワークアラウンド要 |
| FastAPI | ネイティブ動作 | ASGI ミドルウェア経由の“橋渡し” |
| 閉域（VNet / Private Endpoint） | 対応 | Flex Consumption で対応 |
| コスト | 常時課金 | 散発トラフィックで安価 |

**要点（2026 一次情報）**

- App Service は継続的な可用性が要る Web アプリ／API のホスティングに最適、Functions はトリガー時のみ動くイベント駆動・サーバーレス向け。
- Standard / Premium の App Service プランはゼロスケールせず常に最低 1 インスタンスを保持する（＝コールドスタートが無い）。真のゼロスケールは従量 Functions と Container Apps のみ。
- Function App はコントローラのネイティブ対応がなく、フルの REST API は構築・保守が難しい。Swagger やモデルバインディングのサポートも限定的でワークアラウンドが要る。
- App Service for Linux では FastAPI が自動検出され ASGI 起動が自動構成される（Python 3.14 以降で有効化、以前は gunicorn + uvicorn ワーカーの起動コマンドが必要）。Functions では FastAPI（ASGI）と標準 HTTP ハンドラの間を ASGI ミドルウェアで橋渡しする必要がある。
- 2026 年、Azure App Service on Linux の Python デプロイのレイテンシが約 30% 短縮、コールドスタート起因のデプロイ失敗（502/503/499）が約 30% 減少。
- 実務の総意は「対立ではなく併用」: App Service がユーザーリクエストを処理し、Functions が背景タスクを非同期処理する。

**本件の結論（`docs/02-architecture/platform-architecture-decision.md` 案A とも整合）**

- メインの同期 REST API（100〜500ms 目標、認可と BigQuery 参照のホットパス）は **App Service（Linux）**。
  - (a) 従量 Functions のゼロスケール由来のコールドスタートが体感レイテンシ SLO を直撃するのを避けられる
  - (b) FastAPI がネイティブに動く
  - (c) 永続プロセスなので BigQuery / SQL の接続プールとプロセス内 LRU（属性キャッシュ）が効く
  - (d) デプロイスロットで Blue/Green ができる
  - ※ レイテンシ要件がある API にとって「常時稼働＝コスト無駄」ではなく「常時稼働＝機能」。
- **Functions は補助に限定**: マテビューのウォーマー（タイマー）、Webhook 受け、将来の iPaaS 非同期連携、案C を採る場合のバッチ同期役。閉域要件は Flex Consumption の VNet / Private Endpoint で満たせる。
- FastAPI を Functions に載せること自体は可能だが、メイン API を Functions に置く積極的理由は本件にはない（コールドスタートと限定的な Web 機能がデメリットとして残る）。

---

---

## 参考（出典）

- Data API builder 概要（対応データソース、CRUD 置き換え、コード不要）: https://learn.microsoft.com/en-us/azure/data-api-builder/overview
- DAB データベースポリシー（OData 述語 → WHERE、@item / @claims）: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/how-to-configure-database-policies
- DAB 認可・ロール（1 リクエスト 1 ロール、X-MS-API-ROLE ヘッダ）: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authorization-overview
- App Service vs Functions（用途の使い分け・併用）: https://www.c-sharpcorner.com/article/azure-app-service-vs-azure-functions-when-to-use-what-in-real-projects/
- ゼロスケールとコールドスタート（Premium は最低 1 インスタンス）: https://cloudwebschool.com/docs/azure/service-comparisons/azure-functions-vs-app-service/
- Function App は full REST API に不向き（コントローラ非対応・Swagger 限定）: https://matthewthomas.cloud/posts/function-app-vs-web-app-for-apis/
- FastAPI on App Service（自動検出・ASGI 自動構成、Python 3.14+）: https://techcommunity.microsoft.com/blog/appsonazureblog/simplifying-fastapi-deployments-on-azure-app-service-for-linux/4520103
- App Service Python デプロイ改善（2026 年 5 月、約 30% 短縮）: https://azure.github.io/AppService/2026/05/18/platform-improvements-for-python-ai-apps-on-azure-app-service.html
- FastAPI on Functions（ASGI ミドルウェアで橋渡し）: https://devblogs.microsoft.com/cosmosdb/building-your-first-serverless-http-api-on-azure-with-azure-functions-fastapi/
- マルチテナントでの App Service / Functions（Flex Consumption の VNet / Private Endpoint）: https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/app-service
