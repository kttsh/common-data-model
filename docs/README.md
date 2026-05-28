# Documentation Index

この `docs` 配下は、要件定義から設計詳細化へ進めやすいように、文書の役割で分けています。ファイル名とフォルダ名は英語、本文は現状の日本語メモを活かしています。

## 01 Requirements

- `01-requirements/product-requirements.md`: 目的、スコープ、前提、未確定論点の基準文書。

## 02 Architecture

- `02-architecture/platform-architecture-decision.md`: 採用済みの基盤アーキテクチャ決定。
- `02-architecture/data-api-builder-assessment.md`: DAB 不採用と App Service / Functions の使い分け。
- `02-architecture/repository-structure-options.md`: 候補ランタイム別のリポジトリ構造案。

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
