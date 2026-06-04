# Dataform リポジトリ運用設計（GCP ネイティブ案・不採用）

> ⚠️ **不採用（記録として保持）**: 本書が示す **GCP ネイティブ方式**（release / workflow configuration＋strict act-as カスタム SA＋Terraform）は、その後の検討で **CLI 路線へ変更されたため採用しない**。Dataform のデプロイ運用の正は **`../02-architecture/repository-strategy.md` §8（CLI 方式）と `migration-plan.md`（CLI 路線の移行手順）** を参照すること。本書は、GCP ネイティブ案を検討した記録として残す（将来 Container/native 運用を再評価する際の比較材料）。

`../02-architecture/repository-strategy.md` の Dataform 部分を、2026 年の GCP ネイティブ標準（BigQuery 統合・release / workflow 構成・strict act-as）に合わせて検討したもの。**当初は本方式を確定としたが、後に CLI 路線へ反転した**（経緯は `migration-plan.md` 冒頭の「旧前提からの変更」を参照）。

- 作成日: 2026-06-03
- ステータス: **不採用（GCP ネイティブ案）**。デプロイは CLI 路線に変更（最新は `../02-architecture/repository-strategy.md` §8・`migration-plan.md`）
- 関連: `../02-architecture/repository-strategy.md`（案 B・全体方針）、`dataform-naming-convention.md`（命名・層構造）、`migration-plan.md`（移行ステップ＝CLI 路線）
- 旧設計からの前提変更（※本書が検討した GCP ネイティブ案の内部での差分。CLI 路線への反転は上記バナー参照）:
  - 旧 `environments/*.json`（コード同梱の環境定義）方式は **廃止** → **release configuration**（Terraform 管理推奨）へ移譲。
  - CLI / Docker デプロイ案・REST `writeFile` 方式は **不採用** → **GCP ネイティブ（release / workflow 構成）** に統一。
  - 環境別リポジトリコピー案 → **単一リポジトリ＋プロジェクトオーバーライド**（option a）に簡素化。

---

## 1. 位置づけと決定事項

> 本節は **GCP ネイティブ案の検討時点の決定**を記録したもの。最終的にはこの方式は採らず CLI 路線に反転している（冒頭バナー／`migration-plan.md` 参照）。

`../02-architecture/repository-strategy.md` の案 B（プラットフォーム別分割、Dataform は専用 Git リポジトリを持つ）を踏襲し、その Dataform の **デプロイ／環境／昇格の具体方式を GCP ネイティブで検討**した（当時の暫定決定）。検討時点の決定は次のとおり。

| # | 論点 | 決定 |
|---|---|---|
| 1 | マルチ環境の作り方 | **単一 Dataform リポジトリ＋ release configuration のプロジェクトオーバーライド**で dev/stg/prod を表現（option a に簡素化） |
| 2 | stg→prod の承認ゲート | **保留**。確定するまでは §7 の暫定運用（手動で固定 commit を前進）で回す |
| 3 | スケジューリング | **保留だがシンプル維持**。ネイティブの workflow configuration による定期 run のみ。高度な DAG は将来課題 |

---

## 2. 2026 のネイティブ標準（前提）

- Dataform は BigQuery / BigQuery Studio に統合され、SQL 変換を依存グラフ・Git・テスト・ドキュメント付きで扱う。Dataform のワークフローは **BigQuery pipelines**・data preparations・ノートブック・保存クエリの基盤になっている。
- ライフサイクルは 3 部品:
  - **development workspace ＋ workspace compilation overrides** … 開発の隔離（開発者ごとの schema suffix 等。手動実行にのみ適用）。
  - **release configuration** … リポジトリのコンパイル結果テンプレート。Git の commitish（branch / tag / commit SHA）に紐づけ、**GCP プロジェクト・schema suffix・table prefix・vars を上書き**できる。
  - **workflow configuration** … 選択した release configuration の最新コンパイル結果を、スケジュールに従って BigQuery に実行する。
- **前提条件**: いずれの方式も「**単一の Dataform リポジトリ ＝ 単一のリモート Git リポジトリ**」を要求する。テーブル別・環境別ブランチは不要。
- **strict act-as mode が全リポジトリで強制**。実行にはカスタムサービスアカウント（または個人のユーザー認証）が必須（§4）。

---

## 3. アーキテクチャ（単一リポジトリ × プロジェクトオーバーライド）

```
            ┌──────────────────────────────────────────────┐
GitHub      │  ai-driven-comod-cdem  (main, trunk-based)     │  ← 単一 Git リポジトリ
(remote)    └──────────────────────────────────────────────┘
                              │ Git 接続（native: Connect to Git）
                              ▼
            ┌──────────────────────────────────────────────┐
Dataform    │  Dataform repository × 1                       │  ← 管理 PJ もしくは pj-dev に1つだけ作成
(GCP)       │  workflow_settings.yaml は環境非依存             │
            └──────────────────────────────────────────────┘
                 │ release configuration（compilation override: project）
     ┌───────────┼─────────────┬────────────────────┐
     ▼           ▼             ▼                     │ workspace compilation override
  release:dev  release:stg  release:prod             │ （開発者ごと schema suffix）
  proj=pj-dev  proj=pj-stg  proj=pj-prod             ▼
     │           │             │              development workspaces
     ▼           ▼             ▼              （feature ブランチ追跡・手動 run）
  BigQuery     BigQuery      BigQuery
  pj-dev       pj-stg        pj-prod
   ▲             ▲             ▲
   └──── workflow configuration（定期 run・§8）────┘
```

- Dataform リポジトリは **1 つだけ**作成し、単一の GitHub リポジトリ（`ai-driven-comod-cdem`）に接続する。リポジトリの置き場所は専用の管理プロジェクト推奨（プロジェクト数を抑えたいなら `pj-dev` でも可）。
- `workflow_settings.yaml` は **環境非依存**。`defaultProject` は基準値（dev 相当）に置き、環境差は release configuration の **プロジェクトオーバーライド**で材質化先を切り替える。
- 開発の隔離は workspace compilation overrides（開発者ごとの schema suffix）で行う。

### workflow_settings.yaml（環境非依存・例）

```yaml
defaultProject: pj-dev                 # 基準プロジェクト。環境差は release config で上書き
defaultDataset: dataform               # 既定 dataset（層は config.schema / suffix で制御）
defaultLocation: asia-northeast1       # リージョン（公開データセット混在時は整合に注意）
defaultAssertionDataset: dataform_assertions
dataformCoreVersion: <検証済みバージョンを明示ピン>   # 互換性回帰回避のため必ず固定
vars:
  # 共通変数のみ。環境名（dev/stg/prod）やプロジェクト ID はハードコードしない
```

### release configuration（GCP 側設定・概念。リポジトリ内ファイルではない）

| release | git commitish | project override | schema suffix | 役割 |
|---|---|---|---|---|
| `dev`  | `main`                   | `pj-dev`  | （無し） | main を継続コンパイル → dev へ |
| `stg`  | `<昇格対象の commit/tag>` | `pj-stg`  | （任意） | dev 検証済みの同一 commit を stg へ |
| `prod` | `<昇格対象の commit/tag>` | `pj-prod` | （無し） | stg 検証済みの同一 commit を prod へ |

> release configuration・workflow configuration・カスタム SA・IAM は **Terraform で管理**して config-as-code を保つ（旧 `environments/*.json` の役割をここが引き継ぐ）。

---

## 4. 認証・IAM（strict act-as 必須）

strict act-as mode は全 Dataform リポジトリで強制され、ワークフロー実行にはカスタム SA が必須。既存リポジトリは 2026-04-29〜07-31 にかけて段階適用されるため、**期限内の対応が必要**（未対応だと既定サービスエージェントで動いていた実行が停止する）。

設定:

1. 環境ごとに **カスタム SA** を用意（例 `sa-dataform-dev@pj-dev` / `sa-dataform-stg@pj-stg` / `sa-dataform-prd@pj-prod`）。ワークフローはこのカスタム SA で実行する。
2. **デフォルト Dataform サービスエージェント**（`service-PROJECT_NUMBER@gcp-sa-dataform.iam.gserviceaccount.com`）に、各カスタム SA への **Service Account User** と **Service Account Token Creator**（actAs）を付与。これにより既定エージェントがカスタム SA をインパーソネートして実行する。
3. option a は **クロスプロジェクト**（リポジトリのプロジェクト ≠ 材質化先プロジェクト）になるため、**クロスプロジェクトのサービスアカウント・アタッチ権限**を明示付与する。
4. 各カスタム SA に、材質化先 BigQuery への最小権限（例 `roles/bigquery.dataEditor` ＋ `roles/bigquery.jobUser`）を付与。
5. CI（GitHub Actions）→ GCP は **OIDC / Workload Identity Federation（鍵レス）**。**SA の JSON キーは廃止**。
6. GitHub リモートが組織ポリシー `dataform.restrictGitRemotes` の許可リストに無い場合は、接続前に allow-list へ追加する。

---

## 5. リポジトリ構成（命名規約準拠）

`dataform-naming-convention.md` に準拠（小文字 snake_case／ファイル名＝テーブル名／`sources` はソース名プレフィックス／`intermediate` は `stg_`／`outputs` は簡潔名・`dim_`/`fct_` 不使用）。**環境差ファイルは置かない**。

```
ai-driven-comod-cdem/
├── definitions/
│   ├── sources/                   # declaration ＋軽い変換
│   │   └── sap/
│   │       └── coep.sqlx          # 旧 V_COEP の素（※層は要確認・移行ドキュメント参照）
│   ├── intermediate/              # 結合・本格変換（API 非公開）
│   │   └── stg_*.sqlx
│   └── outputs/                   # 出力＝API 公開面（ドメイン単位）
│       ├── cost/
│       │   ├── cost_master.sqlx   # 旧 Dim_COST_MASTER
│       │   └── cost_detail.sqlx   # 旧 Fact_COST_DETAIL
│       └── order/
│           ├── order_master.sqlx  # 旧 Dim_ORDER_MASTER
│           └── order_detail.sqlx  # 旧 Fact_ORDER_DETAIL
├── includes/                      # 共通ヘルパ（増やさない）
│   ├── constants.js
│   ├── naming.js
│   └── assertions.js
├── workflow_settings.yaml         # ルートに1本だけ・環境非依存
├── package.json
└── .github/workflows/
    └── ci.yml                     # PR 検証のみ（compile + dry-run + assertions）
```

> 旧設計にあった `environments/` ディレクトリは **作らない**。環境差は release configuration（§3）に置く。

---

## 6. デプロイ・昇格モデル（build-once ＝ 同一 git commit）

- `main` マージ → `dev` release configuration が `main` をコンパイル（自動 or 手動）→ workflow configuration が **pj-dev** に run → 動作確認。
- 検証後、**同一 git commit（タグ）**を `stg` release configuration の commitish に固定 → run → 確認 → 同じ commit を `prod` に固定 → run。
- 昇格キーは **git SHA**。Dataform のコンパイルは hermetic（同じコード→同じ SQL）なので、各環境で再コンパイルしても再現性は担保される（＝ネイティブ流の build-once）。環境差は **オーバーライドのみ**。
- **fix-forward**: stg / prod の BigQuery テーブルや設定を手で直さない。直すのは常に `main`。修正は dev で再現確認してから昇格し直す。

```
feature → PR(CI: compile + dry-run + assertions) → main
   main          →  dev release/​workflow が run（pj-dev）→ 確認
   同一 commit    →  stg release を当該 commit に固定 → run（pj-stg）→ 確認
   同一 commit    →  prod release を当該 commit に固定 → run（pj-prod）→ tag 付与
```

---

## 7. 承認ゲート（保留中の暫定運用）

正式な承認ゲート方式は **保留**（§10 オープン事項）。確定までの暫定運用:

- `prod` release configuration の固定 commit / tag を、**stg 検証完了後に手動で前進**させる。
- prod 到達コミットに `dataform-vX.Y.Z` タグを付与（人間可読エイリアス）。
- 操作は限られた担当者に限定し、変更は Git 履歴に残す（誰がどの commit を prod に上げたかが追える）。

将来の確定候補（決め打ちしない）:

- (i) 上記の **手動前進**を正式運用化する。
- (ii) **GitHub Environments の required reviewers** 付きジョブから Dataform API で prod release/workflow を更新・起動する（承認を CI 側に持たせる）。

---

## 8. スケジューリング（シンプル維持）

- ネイティブの **workflow configuration** で定期 run のみ（例: 日次）。release configuration を選び、実行アクション（`tag` フィルタ）と頻度・タイムゾーンを設定する。
- 当面はこれ以上作り込まない。他サービスをまたぐ依存や複雑な DAG が必要になった段階で **Managed Service for Apache Airflow** / **Workflows** へ拡張する（将来課題）。
- スケジュール頻度・tag 設計はデプロイとは別決定として後で詰める。

---

## 9. CI（GitHub Actions は検証のみ）

- `ci.yml`（PR トリガー）: `dataform compile` ＋ dry-run ＋ assertions。**デプロイはしない**（デプロイは release / workflow 構成が担当）。
- 認証は OIDC / WIF。
- `main` 保護・squash・トランクベース。ブランチ運用の詳細は `dataform-migration-plan.md` および §11 を参照。

---

## 10. オープン事項

- [ ] 承認ゲートの正式方式（手動前進 / GitHub Actions＋Dataform API）。
- [ ] スケジュール頻度・`tag` 設計。
- [ ] `V_COEP` の層（`sources` の declaration か `intermediate/stg_coep` か）の確定。
- [ ] Terraform 化の範囲（repository・release/workflow configuration・カスタム SA・IAM）。
- [ ] 下流（API `repository.py` / BI）の有無に応じたリネーム方式（移行ドキュメント §6）。

---

## 11. 旧設計からの変更点（差分サマリ）

| 項目 | 旧（`repository-strategy.md` 等） | 新（本書） |
|---|---|---|
| 環境差の吸収 | `environments/*.json` ＋ CLI | release configuration の compilation override（Terraform 管理） |
| デプロイ手段 | CLI / Docker、または REST `writeFile` | GCP ネイティブ release / workflow 構成 |
| マルチ環境 | 環境別リポジトリコピー（案の一つ） | 単一リポジトリ＋プロジェクトオーバーライド（option a） |
| 認証 | SA JSON キー／既定サービスエージェント | カスタム SA ＋ actAs インパーソネーション、CI は OIDC/WIF |
| 環境書き換え | `dataform-rewrite.sh`（`sed`） | 廃止 |
| ブランチ | テーブル別×環境別 | トランクベース（単一 `main`） |

---

## 参考（一次情報）

- Best practices for the workflow lifecycle — https://cloud.google.com/dataform/docs/managing-code-lifecycle
- Configure compilations（release configuration / workspace compilation overrides）— https://cloud.google.com/dataform/docs/configure-compilation
- Schedule runs（workflow configurations）— https://cloud.google.com/dataform/docs/workflow-configurations
- Use strict act-as mode — https://cloud.google.com/dataform/docs/strict-act-as-mode
- Control access with IAM — https://cloud.google.com/dataform/docs/access-control
- Create a repository（custom SA・restrictGitRemotes）— https://cloud.google.com/dataform/docs/create-repository
- Introduction to BigQuery pipelines — https://cloud.google.com/bigquery/docs/pipelines-introduction
- Introduction to data transformation — https://cloud.google.com/bigquery/docs/transform-intro
