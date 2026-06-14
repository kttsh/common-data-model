# Common Data Model API Platform

BigQuery 上の基幹システムデータから共通データモデルを構築し、社内の各ビジネスユニットが REST API として利用できるようにするための要件・設計リポジトリです。

## Compass

このリポジトリの羅針盤は、次の順で読むと全体像がつかめます。

1. `docs/01-requirements/product-requirements.md` で目的、スコープ、制約を確認する。
2. `docs/02-architecture/platform-architecture-decision.md` で採用済みアーキテクチャを確認する。
3. `docs/03-authorization/authorization-strategy.md` で権限制御の方針を確認する。
4. `docs/05-planning/implementation-roadmap.md` で次に詳細化すべき設計・PoC を確認する。

## Current Baseline

- データソースは BigQuery。
- API 実行層は ARO（Azure Red Hat OpenShift）上のコンテナを中心にし、既存共有の Azure API Management を入口にする（2026-06 に App Service から切替）。
- キャッシュは APIM 組込みキャッシュと コンテナ（Pod）内 LRU を起点にする。
- 認証は Entra ID。認可属性は Open-GIM / オンプレ SQL Server 側の属性ストアを参照する。
- 行・列レベル制御は API 層の ABAC を中心にし、BigQuery にはパラメータ化 WHERE を押し下げる。
- DAB は BigQuery 非対応と認可モデル不一致により採用しない。

## Open Design Topics

- Proxy 仕様と BigQuery クライアント疎通。
- ARO クラスタ構成（ノード SKU・HPA・private cluster）とデプロイ戦略（Route 切替の blue/green 等）、APIM との接続方式。
- APIM の Product / Subscription / 命名規約。
- Open-GIM の属性項目、突合キー、到達経路。
- API ランタイム・フレームワーク最終選定。
- 認可ポリシー形式、`authorize()` の戻り値、AST-to-BigQuery SQL 翻訳。
- 監査ログスキーマと SLO。

## Documentation Map

詳細な文書一覧は `docs/README.md` を参照してください。
