# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## このリポジトリの性質

**設計・要件ドキュメントのみのリポジトリ**（コードなし。追跡ファイルはすべて Markdown）。
BigQuery 上の基幹データから「共通データモデル」を作り、社内ビジネスユニットへ **REST API** として提供する基盤の、要件定義から設計詳細化までを管理する。実装コード（Dataform / FastAPI / 認可ポリシー）は別リポジトリで持つ前提で、ここはその設計の正本。

- ビルド・テスト・lint・パッケージマネージャは**存在しない**。これらのコマンドを探さないこと。作業は Markdown の読み書きが中心。
- 本文は**日本語**、ファイル名・フォルダ名は**英語**。この規約を維持する。
- 利用者への応答も日本語。

## 最重要の編集規約: 「正本（owner）」原則

各トピックは**正本ドキュメントを 1 つだけ**持ち、他文書は「要点 + リンク」で参照する。**重複記述を増やさないことが、この文書群の設計思想**。編集時は必ず以下を守る:

- ある事実・決定を書く前に、その**トピックの正本がどれか**を確認する（下表）。正本以外には実体を書かず、正本へリンクする。
- 用語・略語は `docs/03-authorization/glossary.md` が正本。各認可文書は個別の略語表を持たない。新語はここだけに足す。
- **04-research は調査の生記録**であり、**決定はしない**。決定は 02-architecture / 03-authorization 側が正。research に決定を書かない。
- ドキュメント間参照は相対パス（`../02-architecture/...`）で書く。クロス参照を本文に乱立させず、リンク集や脚注にまとめる傾向に合わせる。

### トピック → 正本の対応

| トピック | 正本 |
|---|---|
| 用語・略語 | `docs/03-authorization/glossary.md` |
| 基盤アーキテクチャ決定（案A） | `docs/02-architecture/platform-architecture-decision.md` |
| 言語・FW 決定（Python/FastAPI） | `docs/02-architecture/runtime-framework-decision.md` |
| FW 比較（生記録） | `docs/04-research/api-runtime-framework-comparison-2026.md` |
| 認可ライブラリ比較・PyCasbin 採用根拠 | `docs/04-research/abac-authz-library-comparison.md` |
| PEP/PDP 責務分担・`authorize()` IF | `docs/03-authorization/authorization-boundaries-and-interface.md` |
| リポジトリ分割戦略・ブランチ・昇格フロー | `docs/02-architecture/repository-strategy.md` |
| Dataform 構成・命名規約 | `docs/06-data-platform/dataform-naming-convention.md` |
| Dataform 移行手順（CLI 路線） | `docs/06-data-platform/migration-plan.md` |

## 文章表現・記法のルール

書き方に迷ったらここに従う（過去のフィードバックの集約）。重複排除・リンク参照・相対パスは上の「正本原則」が正。

- **平易で一般的な言葉を使う**。造語・凝った比喩・きどった言い回しは避け、初見で伝わる表現にする（例: 「2 段の裁き」→「認可判断を 2 段に分ける」）。
- **読み手の前提に深さを合わせる**。社内で周知のコード体系・用語は深掘りせず簡潔に。専門的・込み入った実装詳細は本文に詰め込まず、別紙（補足ドキュメント）へ切り出す。
- **解説ドキュメントは「基本の仕組み → 実例 → 補足 → 展開イメージ（例: BigQuery への落とし込み）」を基本構成**にする。書ききれない話題は本文に残さず該当の正本へ飛ばす。
- **本文に節番号レベルの相互参照（`basic-specification.md §8` のような記述）を書かない**。参照は正本へのリンクにとどめ、原則トップの README から辿れれば十分とする。
- **初学者向けの手順・フローは mermaid 図で示す**。文章だけで追わせない。
- **メタ情報（ステータス・Related 等のフロントマター）を各文書に重複させない**。前提は README と本文から読めるようにする。

## ドキュメント構成と読む順序

役割別フォルダで分割。全体像をつかむ推奨順（README の Compass より）:

1. `docs/01-requirements/product-requirements.md` — 目的・スコープ・制約・未確定論点の基準
2. `docs/02-architecture/platform-architecture-decision.md` — 採用済みアーキテクチャ
3. `docs/03-authorization/authorization-strategy.md` — 権限制御の方針
4. `docs/05-planning/implementation-roadmap.md` — 次に詳細化すべき設計・PoC

フォルダの役割: `01-requirements`（要件）/ `02-architecture`（採用決定）/ `03-authorization`（認可設計、`pycasbin/` に実装メモ）/ `04-research`（調査の生記録、決定しない）/ `05-planning`（ロードマップ）/ `06-data-platform`（Dataform 関連）。文書一覧は `docs/README.md`。

## 設計対象システムの全体像（横断把握が要る部分）

複数文書を読まないと見えない「設計中のシステム」の骨格:

```
[利用者/SA] → Azure APIM(既存共有: JWT事前検証/スロットル/組込みキャッシュ)
            → Azure App Service(Linux, Python/FastAPI)
                 ├ 埋め込み PEP/PDP(PyCasbin) ─ Open-GIM へ属性参照(ExpressRoute)
                 ├ プロセス内 LRU キャッシュ
                 └ BigQuery クライアント(Proxy 経由) ← Dataform が作るビュー/MV
```

横断して効く確定事項:

- **アーキテクチャ = 案A**（最小構成: 追加 Azure サービスゼロ）。案B/案C は将来比較用に保持。
- **認可は App Service 内 ABAC**。判断を 2 段に分ける ―― ①粗い gate(allow/deny、deny は 403 で短絡) → ②行フィルタ・列マスク生成。`authorize()` は allow/deny だけでなく **row_filter(AST) と masked_columns を返す**。FastAPI が AST を**パラメータ化 WHERE** に翻訳して BigQuery へ押し下げる。BigQuery RLS は二重管理回避のため使わない。
- **PyCasbin（埋め込み PDP）で確定**。外部 PDP（Cerbos/OPA）は将来オプション。差し替えに備え PEP↔PDP は AuthZEN 1.0 を意識。
- **認可属性の正本は Open-GIM**（社内ユーザーリポジトリ。オンプレ SQL Server と同一実体かは未確定論点）。
- **リポジトリ分割 = 案B**（デプロイ境界 = リポジトリ境界）。`dataform`（→BigQuery）と `api`（FastAPI→App Service）を別リポに。エンティティ（Order/Invoice/…）が増えてもリポは増やさず、各リポ内の**エンティティ単位ディレクトリ**を足す（モジュラーモノリス）。1 エンティティ = URL ↔ `src/<entity>/` ↔ `definitions/outputs/<entity>/` ↔ `policies/<entity>.csv` を同名 1:1 で対応。
- **CI/CD = GitHub Actions**（Harness は採用見送り）。**build-once / promote**（同一 digest・同一 commit を dev→stg→prod）、**fix-forward**（環境を直接いじらず常にコードを直す）、**トランクベース**（main 保護 + squash + 承認ゲート）。
- **Dataform デプロイ = CLI 方式で確定**（`@dataform/cli`、環境差は `--default-database` 等で注入）。GCP ネイティブ運用は不採用。
- 主要な社内制約: Container Apps/AKS 不可（App Service / Functions のみ）、Redis 系不可、Azure 永続ストアは Azure SQL のみ承認、外部通信は Proxy 経由のみ。

## 主な未確定論点（追跡対象）

設計を進める前提として残っている: Proxy 仕様と BigQuery クライアント疎通（最優先・PoC で実機検証）、App Service プラン制約、APIM の Product/Subscription 運用ルール、Open-GIM のスキーマ・突合キー・到達経路、AST→BigQuery SQL 翻訳層の設計、監査ログスキーマと SLO。詳細は各文書末尾の「未決事項 / 次アクション」表。

## 既知の不整合

`docs/README.md` は `02-architecture/data-api-builder-assessment.md` と `02-architecture/repository-structure-options.md` を参照しているが、**両ファイルは現在リポジトリに存在しない**（DAB 不採用の正本は実体なし。リンク先を追加または参照を修正する余地あり）。文書を編集する際はこの種のリンク切れに注意する。
