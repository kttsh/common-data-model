# Documentation Index

この `docs` 配下は、要件定義から設計詳細化へ進めやすいように、文書の役割で分けています。ファイル名とフォルダ名は英語、本文は現状の日本語メモを活かしています。

**正本の考え方**: 各トピックは「正本（owner）」を 1 つ決め、他文書は要点＋リンクで参照します（重複記述を増やさない）。とくに **04 Research は調査の生記録**で、決定は 02 Architecture / 03 Authorization 側が正です。用語は `03-authorization/glossary.md`、認可ライブラリ比較・PyCasbin 採用は `04-research/abac-authz-library-comparison.md` に集約しています。

## 01 Requirements

- `01-requirements/product-requirements.md`: 目的、スコープ、前提、未確定論点の基準文書。

## 02 Architecture

- `02-architecture/platform-architecture-decision.md`: 採用済みの基盤アーキテクチャ決定。
- `02-architecture/runtime-framework-decision.md`: 言語・FW の採用決定（**Python / FastAPI**、D1 クローズ）と 2026 年情報に基づく選定理由。
- `02-architecture/data-api-builder-assessment.md`: DAB 不採用と App Service / Functions の使い分け。
- `02-architecture/repository-structure-options.md`: 候補ランタイム別のリポジトリ構造案（採用案は #2 Python / FastAPI）。

## 03 Authorization

- `03-authorization/glossary.md`: 認可ドキュメント共通の略語・用語集（**用語の正本**。各認可文書はここを参照し、個別の略語表は持たない）。
- `03-authorization/authorization-strategy.md`: 権限制御の論点整理と技術方針。
- `03-authorization/authorization-boundaries-and-interface.md`: PEP/PDP 責務分担と `authorize()` インターフェースの決定メモ。
- `03-authorization/pycasbin/basic-specification.md`: PyCasbin の基本仕様、対応モデル、FastAPI/PDP 内で担う範囲の調査メモ。
- `03-authorization/pycasbin/policy-examples-purchase-order.md`: 注文伝票を題材にした PyCasbin ポリシー定義例（居住国・部門・役職・列マスクなど）。「基本的な仕組み → 実例 → 補足 → BQ 展開イメージ」の順で、特性のあるコードの捌き方とルール定義方法に集中。
- `03-authorization/pycasbin/row-scope-to-bigquery-implementation.md`: 上記の別紙。`row_scope` を BigQuery の WHERE に展開する Python 実装（前方一致の範囲展開・パラメータ集約・fail-safe）。
- `03-authorization/pycasbin/single-record-final-check.md`: 上記の別紙。詳細画面で 1 件開いたときの最終チェック（1 件単位の `enforce()`、time-of-check / time-of-use のずれ対策）。
- `03-authorization/row-level-filtering-layering.md`: 行レベル絞り込みをどの層で担うかの整理。
- `03-authorization/shared-pdp-across-api-and-bigquery.md`: PDP を FastAPI 経路と BigQuery を叩く処理で共用（統一）するための検討メモ。

調査の生記録。比較表・出典を持つが決定はしない（決定は 02・03 が正）。

- `04-research/api-runtime-framework-comparison-2026.md`: API ランタイム・フレームワーク比較（**FW 比較の正本**。決定は `runtime-framework-decision.md`）。
- `04-research/abac-authz-library-comparison.md`: ABAC 認可ライブラリ比較と **PyCasbin 採用決定の正本**。他文書のライブラリ比較表・採用根拠はここに集約。
- `04-research/pep-pdp-design-research-2026.md`: PEP/PDP 設計の世の中のプラクティスと多エンジンのポリシー記述例（比較・将来オプション）。
- `04-research/ci-cd-delivery-research-2026.md`: CI/CD、Harness、SBOM、STO、SRM の調査（**Harness は採用見送り**。実装は GitHub Actions）。
- `04-research/authorization-models-and-standards-2026.md`: 認可モデル、標準（AuthZEN/RFC）、FastAPI 実装プラクティス調査。

## 05 Planning

- `05-planning/implementation-roadmap.md`: 直近の設計作業、PoC 範囲、調査バックログ。

## 06 Data Platform

API の元ネタとなる Dataform（→ BigQuery）のリポジトリ戦略・命名・運用・移行をまとめたトピック集約。

- `02-architecture/repository-strategy.md`: プラットフォーム全体のリポジトリ分割戦略（案 B 確定）・昇格フロー・ブランチ戦略。**Dataform デプロイは CLI 方式で確定**（02-architecture 配下に配置）。
- `06-data-platform/dataform-naming-convention.md`: Dataform の構成・命名規約リファレンス（`sources`/`intermediate`/`outputs` 準拠）。
- `06-data-platform/dataform-operating-model.md`: Dataform の GCP ネイティブ運用設計の検討記録。**不採用**（CLI 路線へ変更されたため。最新は `repository-strategy.md`・`migration-plan.md`）。
- `06-data-platform/migration-plan.md`: あるべき姿（CLI 路線）への移行手順・トランクベースのブランチステップ。
