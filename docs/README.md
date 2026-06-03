# Documentation Index

この `docs` 配下は、要件定義から設計詳細化へ進めやすいように、文書の役割で分けています。ファイル名とフォルダ名は英語、本文は現状の日本語メモを活かしています。

## 01 Requirements

- `01-requirements/product-requirements.md`: 目的、スコープ、前提、未確定論点の基準文書。

## 02 Architecture

- `02-architecture/platform-architecture-decision.md`: 採用済みの基盤アーキテクチャ決定。
- `02-architecture/runtime-framework-decision.md`: 言語・FW の採用決定（**Python / FastAPI**、D1 クローズ）と 2026 年情報に基づく選定理由。
- `02-architecture/data-api-builder-assessment.md`: DAB 不採用と App Service / Functions の使い分け。
- `02-architecture/repository-structure-options.md`: 候補ランタイム別のリポジトリ構造案（採用案は #2 Python / FastAPI）。

## 03 Authorization

- `03-authorization/authorization-strategy.md`: 権限制御の論点整理と技術方針。
- `03-authorization/authorization-boundaries-and-interface.md`: PEP/PDP 責務分担と `authorize()` インターフェースの決定メモ。
- `03-authorization/row-level-filtering-layering.md`: 行レベル絞り込みをどの層で担うかの整理。

## 04 Research

- `04-research/api-runtime-framework-comparison-2026.md`: API ランタイム・フレームワーク比較。
- `04-research/ci-cd-delivery-research-2026.md`: CI/CD、Harness、SBOM、STO、SRM の調査。
- `04-research/authorization-models-and-standards-2026.md`: 認可モデル、標準、FastAPI 実装プラクティス調査。

## 05 Planning

- `05-planning/implementation-roadmap.md`: 直近の設計作業、PoC 範囲、調査バックログ。

## 06 Data Platform

API の元ネタとなる Dataform（→ BigQuery）のリポジトリ戦略・命名・運用・移行をまとめたトピック集約。

- `06-data-platform/repository-strategy.md`: プラットフォーム全体のリポジトリ分割戦略（案 B 確定）・昇格フロー・ブランチ戦略。
- `06-data-platform/dataform-naming-convention.md`: Dataform の構成・命名規約リファレンス（`sources`/`intermediate`/`outputs` 準拠）。
- `06-data-platform/dataform-operating-model.md`: Dataform の GCP ネイティブ運用設計（単一リポジトリ＋release configuration オーバーライド）。到達点。
- `06-data-platform/dataform-migration-plan.md`: あるべき姿への移行手順・トランクベースのブランチステップ。
