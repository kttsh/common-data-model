# BigQuery / Dataform データ基盤 設計ルール
 
最終更新: 2026-06-11
 
---
 
## 1. 全体構造 — 3層の分離
 
本基盤は「実体」「インフラ定義」「変換定義」の3層で構成する。GCP上のフォルダ／データセットはファイルを格納する箱ではなく、GitHub上のコードを適用した結果として実体（テーブル）が生成される。
 
| 層 | 実体 | 管理場所 | 実行主体 |
|---|---|---|---|
| ① GCP階層（フォルダ/プロジェクト/データセット/テーブル) | BigQueryのテーブル・ビュー | GCP上（②③の出力結果。直接は変更しない） | — |
| ② インフラ定義 | `.tf` ファイル | GitHubリポジトリ（インフラ用） | Terraform（初回ローカル → 以降CI） |
| ③ 変換定義 | `.sqlx` / `.yaml` ファイル | GitHubリポジトリ（Dataform用） | Dataform（compile & execute） |
 
- ②と③は**別リポジトリ**とする。Dataformは `workflow_settings.yaml` のリポジトリ直下配置が必須のため、インフラコードと同居させない。
- Terraformの責務は「器を作る」こと（フォルダ・プロジェクト・データセット・Dataformのrepository / release config / workflow config）。**変換SQLの実行はDataformの責務**であり、`terraform apply` では変換は流れない。
- Terraformのローカル実行はステートバックエンド構築等のブートストラップ時のみ。定常運用はGitHub Actions + OIDC（Workload Identity Federation）でCIから実行する。
 
## 2. GCPリソース階層
 
### フォルダ（2軸）
 
| フォルダ種別 | 数 | データレイヤー | 役割 |
|---|---|---|---|
| 連携元フォルダ | 源泉システムごとに1つ（SAP、Coupa…） | **Bronzeのみ** | 源泉の生データ着地。フォルダ分割によりGCP内のデータ再利用性を向上 |
| 共通データモデルフォルダ | 1つ | Silver〜Gold | すべての共通データモデルを管理。APIでアウトプットするテーブルのみを対象とする |
 
### プロジェクト（環境3面）
 
1フォルダにつき、環境単位で3プロジェクトに分割する。
 
| 環境 | 目的 | 権限 |
|---|---|---|
| 本番 | 業務利用。外部システム連携・データ提供APIのバックエンド | 厳格に制御。直接書き込み禁止。Pull Request経由で変更 |
| 検証 | 本番前の最終確認。データ品質チェック・ワークフロー検証・性能検証 | 開発者 + 一部レビュー担当者。書き込み制限あり |
| 開発 | 開発・試行・動作確認。SQL/パイプライン/API開発 | 開発者に広く付与。書き込み・削除も許可 |
 
### データセット
 
データセットはプロジェクト直下にフラットに並ぶ（ネスト不可）。共通データモデルプロジェクトの構成:
 
```
cdm-<env>
├─ intermediate          ← Silver層: dim_*, fct_*
├─ outputs               ← Gold層:  業務名のみ（接頭辞なし）
└─ dataform_assertions   ← データ品質チェック結果
```
 
Bronzeは共通モデルプロジェクトには存在しない。源泉プロジェクト側にあり、Dataformの `type: "declaration"` 経由で参照する（宣言の仕組みと記述例は §4.1 参照）。
 
## 3. データレイヤー定義
 
| 概念レイヤー | 物理データセット名 | 内容 | テーブル命名 |
|---|---|---|---|
| Bronze | `bronze`（源泉プロジェクト側） | 源泉の生データ | `<source>_<table>_raw` |
| Silver | `intermediate` | スタースキーマ。高度な分析およびGoldのソースとして利用（例: 部門費共通モデル） | `fct_xxx`, `dim_xxx` |
| Gold | `outputs` | エンドユーザー提供用フラットテーブル（例: コスト管理モデル）。APIが参照する最終テーブル | 業務名のみ（例: `cost_management`） |
 
- Silver=分析者・Goldの素材、Gold=エンドユーザー提供、という役割分担をUX基準で定義する。
- **命名規約（確定）:** 接頭辞は「データセット名が運ばない情報」を運ぶ場合のみ付与する。`dim_`/`fct_` はスタースキーマ内の役割（ファクト／ディメンションの結合方向）を示し、`intermediate` 内に両者が混在するため必要。Goldは `outputs` というデータセット名が層（=全テーブルがフラット）を既に表しているため接頭辞を付けず、業務語彙のみとする。これによりAPIエンドポイントとの対応（例: `/cost-management` ↔ `outputs.cost_management`）も一貫する。調査経緯は付録参照。
- 本構成のSilver（スタースキーマ）は古典的メダリオンのSilver（洗浄済み原子データ）より1段加工が進んでいる。洗浄工程の置き場所は未決事項（§6-1）。
 
## 4. リポジトリ構成
 
リポジトリは2つ（Dataform用・Terraform用）に分離する（理由は §1 参照）。
 
### 4.1 Dataformリポジトリ
 
```
common-data-model/
├─ workflow_settings.yaml      # defaultProject / defaultLocation / defaultDataset / dataformCoreVersion / vars
├─ package.json
├─ definitions/
│   ├─ sources/                # 源泉Bronzeの宣言（type: declaration）
│   │   ├─ sap/
│   │   └─ coupa/
│   ├─ intermediate/           # Silver: dim_*, fct_* の .sqlx
│   └─ outputs/                # Gold: 業務名の .sqlx（接頭辞なし）
└─ includes/                   # 再利用JS関数・共通SQL断片
```
 
#### 源泉宣言（`type: "declaration"`）の詳細
 
`type: "declaration"` のファイルは**テーブルを作らない**。役割は、Dataform管理外の既存テーブル（源泉プロジェクトのBronze）をDataformに登録し、`${ref()}` で参照可能にすることに限られる。
 
- 宣言を置くことで源泉テーブルが依存DAGとリネージの**起点**に組み込まれる。下流の `.sqlx` で `${ref("sap_cost_master_raw")}` と書くと、コンパイル時に `sap-<env>.bronze.sap_cost_master_raw` という完全修飾名に展開される。
- 実行対象にはならない（コンパイル時の名前解決のみ）。源泉テーブル自体の作成・更新は連携側パイプラインの責務（§6-9 のオーナーシップと整合）。
- **環境切替の注意:** `database: "sap-prod"` とハードコードすると、リリース構成のproject overrideは宣言の `database` に及ばず、開発環境からも本番源泉を読んでしまう。`database` には `dataform.projectConfig.vars` 経由の変数を使い、リリース構成の `vars` で環境別に上書きする。
 
```sqlx
-- definitions/sources/sap/sap_cost_master_raw.sqlx
config {
  type: "declaration",
  database: dataform.projectConfig.vars.sapProject,  // 環境ごとに sap-dev / sap-stg / sap-prod に切替
  schema: "bronze",
  name: "sap_cost_master_raw",
  description: "SAP 原価マスタ（生データ）"
}
```
 
#### 代表的なファイルの例
 
`workflow_settings.yaml`（リポジトリ直下に必須。開発環境の既定値を記載し、検証/本番はリリース構成で上書き）:
 
```yaml
defaultProject: cdm-dev
defaultLocation: asia-northeast1        # 全データセットで統一（§6-2）
defaultDataset: intermediate
defaultAssertionDataset: dataform_assertions
dataformCoreVersion: 3.0.x              # 固定値でピン留めする
vars:
  sapProject: sap-dev                   # 源泉宣言の database に使用
  coupaProject: coupa-dev
```
 
`definitions/intermediate/dim_department.sqlx`（Silver: ディメンション。洗浄層確定後は `stg_` を経由する形に修正、§6-1）:
 
```sqlx
config {
  type: "table",
  schema: "intermediate",
  description: "部門ディメンション",
  assertions: {
    uniqueKey: ["dept_code"],
    nonNull: ["dept_code", "dept_name"]
  }
}
SELECT DISTINCT dept_code, dept_name
FROM ${ref("sap_cost_master_raw")}
```
 
`definitions/outputs/cost_management.sqlx`（Gold: API提供用フラットテーブル。接頭辞なし）:
 
```sqlx
config {
  type: "table",
  schema: "outputs",
  description: "コスト管理モデル（エンドユーザー提供用）",
  bigquery: {
    partitionBy: "posting_date",        # スキャン量制御（§6-6）
    clusterBy: ["dept_code"]
  }
}
SELECT f.*, d.dept_name
FROM ${ref("fct_department_cost")} f
JOIN ${ref("dim_department")} d USING (dept_code)
```
 
#### 構成ルール
 
1. **フォルダ名 = データセット名 = `config.schema` を一致させる。** テーブルの作成先を決めるのはフォルダ位置ではなく configブロックの `schema` / `database` だが、3者を揃えることで「ファイルパス＝テーブルの居場所」が成立し、可読性とオーナーシップが担保される。
2. **レイヤー軸と源泉軸の2軸はフォルダで物理構造にミラーする。** レイヤーは `definitions/` 直下のフォルダ、源泉は `sources/` 内のサブフォルダで表現する。
3. **環境軸はフォルダに持ち込まない。** `definitions/prod/` のような環境別フォルダは禁止。コードは1セットのみとし、環境はリリース構成のproject overrideで振り分ける（3重管理によるドリフト防止）。
4. 依存関係は `${ref(...)}` で表現し、`bronze → intermediate → outputs` の実行順序をDAGとして自動構築させる。
 
### 4.2 Terraformリポジトリ
 
```
infra-gcp/                          # Dataformリポとは別リポジトリ
├─ README.md
├─ backend.tf                       # GCSステートバックエンド定義
├─ providers.tf                     # google / google-beta プロバイダ、WIF認証
├─ versions.tf                      # Terraform本体・プロバイダのバージョン固定
├─ variables.tf                     # location、環境リスト等の入力変数
├─ folders.tf                       # 連携元フォルダ群・共通データモデルフォルダ
├─ projects.tf                      # フォルダ × 環境3面のプロジェクト
├─ datasets.tf                      # bronze / intermediate / outputs / dataform_assertions
├─ iam.tf                           # クロスプロジェクト権限（Authorized Dataset 等、§6-3）
├─ dataform.tf                      # Dataformの器（repository / release config / workflow config）
└─ modules/
    └─ source_system/               # 源泉システム1式（フォルダ+3プロジェクト+bronze）の再利用モジュール
        ├─ main.tf
        ├─ variables.tf
        └─ outputs.tf
```
 
#### 構成ルール
 
1. 源泉システムの追加は `modules/source_system` の呼び出し追加で行う（ファイルのコピペ増殖をしない）。
2. ロケーションは `variables.tf` の単一変数で全データセットに供給する（§6-2 の単一リージョン統一を構造で強制）。
3. リリース構成・ワークフロー構成（Dataformの器）はすべて `dataform.tf` に集約し、環境差分は `code_compilation_config` のみで表現する。
4. 適用はCI（GitHub Actions + OIDC）から行う。ローカル `apply` はブートストラップ時のみ（§1）。
 
#### 代表的なファイルの例
 
`backend.tf`:
 
```hcl
terraform {
  backend "gcs" {
    bucket = "tfstate-data-platform"
    prefix = "infra-gcp"
  }
}
```
 
`datasets.tf`（共通モデル側の例。環境3面を for_each で展開）:
 
```hcl
resource "google_bigquery_dataset" "intermediate" {
  for_each   = toset(["dev", "stg", "prod"])
  project    = google_project.cdm[each.key].project_id
  dataset_id = "intermediate"
  location   = var.location          # 全データセットで統一（§6-2）
}
```
 
`dataform.tf`（器の定義。SQLXの中身はここには現れない）:
 
```hcl
resource "google_dataform_repository" "cdm" {
  provider = google-beta
  name     = "common-data-model"
  project  = google_project.cdm["prod"].project_id
  region   = var.location
  git_remote_settings {
    url            = "https://github.com/<org>/common-data-model.git"
    default_branch = "main"
    authentication_token_secret_version = var.github_token_secret
  }
}
 
resource "google_dataform_repository_release_config" "prod" {
  provider      = google-beta
  repository    = google_dataform_repository.cdm.name
  project       = google_dataform_repository.cdm.project
  region        = var.location
  name          = "prod"
  git_commitish = "main"              # 保護ブランチの同一コミットを再コンパイル（§6-4）
  code_compilation_config {
    default_database = "cdm-prod"     # project override
    vars = {
      sapProject = "sap-prod"         # 源泉宣言の参照先も環境ごとに切替（§4.1）
    }
  }
}
```
 
## 5. 環境分離とプロモーション
 
- ファイルを環境ごとにコピーしない。1つのDataformリポジトリ内で、ワークスペースのコンパイル上書き（開発）とリリース構成・ワークフロー構成（検証/本番）により実行環境を作り分ける。
- 開発: `workflow_settings.yaml` の `defaultProject: cdm-dev` で各自ワークスペースに展開。
- 検証/本番: リリース構成の `code_compilation_config`（project override）で `cdm-stg` / `cdm-prod` に差し替え。
- リリース構成・ワークフロー構成はTerraformで管理する（`google_dataform_repository_release_config` / `google_dataform_repository_workflow_config`）。
- release configの作成だけではコンパイル結果は生成されない。コンパイル結果の生成（cronまたはAPI）→ workflow configによる実行、という段取りを前提に設計する。
 
---
 
## 6. 未決事項と推奨回答
 
| # | 論点 | 推奨 |
|---|---|---|
| 1 | 洗浄層の置き場所 | 共通モデル側に `stg_*` を置く（下記詳細） |
| 2 | リージョン戦略 | 全プロジェクト・全データセットで単一リージョンに統一 |
| 3 | クロスプロジェクトIAM | Authorized Dataset で源泉Bronzeを共通モデルSAに限定公開 |
| 4 | 環境間プロモーション機構 | Terraform管理のリリース構成 + GitHub Environments承認ゲート |
| 5 | データ品質自動化 | Dataformアサーションを必須化しCIゲートに組み込む |
| 6 | パーティション/コスト制御 | 日付パーティション + クラスタリング + bytes上限クォータ |
| 7 | 履歴管理（SCD） | まずSCD Type1 + 日次スナップショットで開始 |
| 8 | データセット名前空間 | 当面2データセット維持、ドメイン増加時に分割基準を再判定 |
| 9 | レイヤー別オーナーシップ | Bronze=連携チーム、Silver/Gold=共通モデルチームと明文化 |
 
### 6-1. 洗浄層の置き場所（最優先）
 
**現状の矛盾:** 連携元フォルダは「Bronzeのみ」だが、共通モデルフォルダは「連携フォルダのSilverから取得」と記述されており、Bronze→Silverの洗浄（型変換・NULL処理・重複排除）の責務が未定義。
 
**推奨:** 源泉フォルダはBronze専用を維持し、洗浄は共通モデル側に置く。具体的には `definitions/sources/` の declaration 直後に `stg_<source>_<table>.sqlx` を配置し、`intermediate` データセット内（または専用 `staging` データセット）に実体化する。フローは `bronze（宣言）→ stg_（洗浄）→ dim_/fct_ → outputs（業務名）`。
 
理由: 「源泉フォルダ=Bronze専用」という現行思想と一貫し、源泉の構造変化を `stg_` で吸収して dim/fct への波及を防げる。資料の「Silverから取得」の記述は「Bronzeから取得」に修正すること。
 
### 6-2. リージョン戦略
 
BigQueryは異なるロケーション間の直接JOIN・参照が不可。源泉・共通モデル・3環境すべてのデータセットを単一リージョンに統一し、`workflow_settings.yaml` の `defaultLocation` とTerraformの `datasets.tf` で固定する。
 
### 6-3. クロスプロジェクトIAM
 
共通モデルプロジェクトのDataform実行サービスアカウントに対し、源泉プロジェクトの `bronze` データセット単位で読み取りを許可する。プロジェクト全体への `dataViewer` 付与は行わず、Authorized Dataset（またはデータセット単位ACL）で公開範囲を絞る。定義は `iam.tf` に集約。
 
### 6-4. 環境間プロモーション機構
 
dev → 検証 → 本番の昇格は「同一コミットの再コンパイル」で実現する。検証・本番それぞれのリリース構成をTerraformで定義し、`git_commitish` は保護ブランチ（main）を指す。本番適用はGitHub Environmentsの承認ゲートを通す。データセットの手動コピーによる昇格は禁止。
 
### 6-5. データ品質自動化
 
各 `.sqlx` のconfigブロックに一意性・非NULL・行条件のアサーションを定義し、結果を `dataform_assertions` に出力する。検証環境でのアサーション全件パスを本番昇格の前提条件としてCIに組み込む。「検証環境でデータ品質チェックを行う」という方針の実体をここで担保する。
 
### 6-6. パーティション/コスト制御
 
Gold（`outputs` 配下のテーブル）はAPI経由のスキャン量がコストとレイテンシに直結するため、日付列パーティション + 主要フィルタ列のクラスタリングを必須とする。プロジェクト単位で per-user / global の query bytes 上限クォータを設定し暴走を防止する。高頻度参照テーブルはマテリアライズドビュー化を個別判断。
 
### 6-7. 履歴管理（SCD）
 
SAPマスタ系 `dim_*` の履歴要件が確定するまでは、SCD Type1（上書き）+ Bronze側での日次スナップショット保持で開始する。履歴参照要件が確定した時点でType2への移行を判断する。Bronzeの取り込み方式（洗い替え/追記）とセットで決定すること。
 
### 6-8. データセット名前空間
 
当面は `intermediate` / `outputs` の2データセットを維持。ドメイン（部門費・コスト管理以外）が増加しテーブル数が管理限界を超えた時点で、`<layer>_<domain>` 形式への分割を再判定する。先行して分割しない（YAGNI）。
 
### 6-9. レイヤー別オーナーシップ
 
メダリオンは各段階の品質責任の契約である。Bronze=源泉連携チーム（着地データの完全性・再処理可能性）、Silver/Gold=共通データモデルチーム（変換ロジック・提供品質）、と責任主体を文書で明示する。
 
---
 
## 付録: テーブル命名規約の調査要約（2026-06調査、§3の決定根拠）
 
### 世の中のプラクティス（3つの立場）
 
| 立場 | 内容 | 代表 |
|---|---|---|
| 接頭辞標準派 | `stg_` / `int_` / `fct_` / `dim_` で役割（層・モデル種別）を示す。現在の主流。接頭辞は「エンドユーザーがクエリすべきテーブル」と「内部部品」を区別するシグナルであり、マテリアライゼーション方式の宣言ではない。公式のプロジェクト評価ツールが接頭辞なしのモデルをルール違反としてフラグするほど制度化されている | dbt スタイルガイドおよびそのエコシステム |
| 無接頭辞派 | カラムナ型DWHのワイドテーブル化でファクト/ディメンションが同一テーブルに同居するようになり、`dim_` / `fact_` は消えつつあるという観察。エンドユーザー向けの層では技術接頭辞より業務語彙（`customer`、`monthly_sales`）を優先 | ワイドテーブル設計を採るモダンDWH論 |
| 接尾辞派 | DBツールはテーブルをアルファベット順に表示するため、接尾辞（`customer_h` 等）にすると同一ビジネスオブジェクトの関連テーブルが隣接して並ぶという実利。古典的DWH製品も `_D`（Dimension）/`_F`（Fact）等の接尾辞体系 | Data Vault系（Scalefree）、Oracle BI Applications |
 
### 判断基準と本PJへの適用
 
判断基準: **接頭辞が運ぶ情報を、データセット名が既に運んでいないか（冗長性の検査）**。
 
| 接頭辞 | 運ぶ情報 | データセットで代替可能か | 判定 |
|---|---|---|---|
| `dim_` / `fct_` | スタースキーマ内の役割（どれがファクトでどれにJOINするか） | 不可（`intermediate` 内に両者が混在） | 採用 |
| `fla_` | テーブルがフラットであること | 可（`outputs` 内は定義上すべてフラット） | 廃止（冗長） |
| `fac_` 表記 | — | — | `fct_` に変更（世間標準と一致させ、参加者・ツール・コーディングエージェントが前提知識のまま読める表記に統一） |
 
### 決定（案B）
 
- Silver（`intermediate`）: `fct_xxx` / `dim_xxx`
- Gold（`outputs`）: 業務名のみ（例: `cost_management`）。APIエンドポイントとの語彙一貫性を優先
 
 
 
### キャッシュ戦略
 
- BI engine でのBigQuery側のキャッシュ機構と、API層でのキャッシュ機構を組み合わせて、パフォーマンスを最適化する。
 
- データの更新頻度やアクセスパターンに応じて、キャッシュの有効期限を設定する。
 
 
### コスト管理
 
- コスト最適化のため、プロジェクト × データセット × 人単位で一定期間でのクエリーの上限を設定できないか？
 
### Observability
- オブザーバービリティ的な観点で、利用のメトリクスを抽出したい。
 
- APIMで統制する、という方向性であれば、APIM側でメトリクスを抽出できるか確認する。
 
- そうでない場合は、BigQuery側でメトリクスを抽出できるか確認する。
 
 
### 次回のお題
- テストの話
- そろそろマージしたルール初版のドラフト
- 1リポジトリ3環境デプロイをトライしてみた
- BigQuery Sharingをためしてみた
