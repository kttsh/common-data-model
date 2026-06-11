# Dataform 詳細命名リファレンス

`common-data-model` リポジトリ（Dataform → BigQuery）のファイル・テーブル・カラム等の詳細命名規約。**層構造・テーブル接頭辞（`dim_`/`fct_`・業務名）・環境分離の決定は [`../02-architecture/bigquery-dataform-design-rules.md`](../02-architecture/bigquery-dataform-design-rules.md) が正本**で、本書はその下位の詳細リファレンス。

- 作成日: 2026-05-31
- 更新日: 2026-06-11（決定の正本を `bigquery-dataform-design-rules.md` へ移管。Silver=`dim_`/`fct_` 採用・outputs フラット化・リポジトリ名 `common-data-model` に追従）

---

## 1. 大原則

| 原則 | 内容 |
|---|---|
| **小文字 snake_case** | テーブル名・カラム名・ファイル名すべて。大文字は使わない（`ORDER_DETAIL` ✕ → `order_detail` ○） |
| **ファイル名 = テーブル名** | `definitions/.../cost_management.sqlx` → BigQuery テーブル `cost_management`。一致させる |
| **フォルダ名 = データセット名 = `config.schema`** | 「ファイルパス＝テーブルの居場所」を成立させる。源泉宣言のみ `sources/<source_system>/` のサブフォルダで源泉軸を表す |
| **BigQuery 命名規則に準拠** | 全ファイル名が BigQuery のテーブル命名規則に従う必要がある（Dataform の制約） |
| **フォルダ・ブランチで環境を分けない** | 環境差（dev/stg/prod）はデプロイ時に注入する。暫定は CLI コンパイラオプション、本筋は Terraform 管理のリリース構成（正は `../02-architecture/bigquery-dataform-design-rules.md`・`../02-architecture/repository-strategy.md`） |

---

## 2. ディレクトリ構成

```
common-data-model/
├── definitions/
│   ├── sources/                      # ① ソース宣言（declaration）。源泉システム別サブフォルダ
│   │   ├── sap/
│   │   │   └── sap_cost_master_raw.sqlx
│   │   └── coupa/
│   ├── intermediate/                 # ② Silver: dim_*/fct_* をフラットに置く（API非公開）
│   │   ├── dim_department.sqlx
│   │   └── fct_department_cost.sqlx
│   └── outputs/                      # ③ Gold＝API公開面: 業務名の .sqlx をフラットに置く
│       └── cost_management.sqlx
├── includes/                         # 共通 JS/SQLX ヘルパ（増やさない）
│   ├── constants.js                  #   project/dataset を vars 経由で参照
│   ├── naming.js                     #   命名規約マクロ
│   └── assertions.js                 #   共通アサーション
├── workflow_settings.yaml            # 環境非依存: defaultProject/Dataset/Location, dataformCoreVersion, vars
├── package.json
└── .github/workflows/
```

`definitions/` 直下は **ワークフローの段階（sources → intermediate → outputs）** で 3 分割し、`intermediate` / `outputs` の中はサブディレクトリを切らずフラットに置く（フォルダ名 = データセット名）。テーブルが増えても `.sqlx` ファイルを足すだけでフォルダ構成は変えない。

---

## 3. 層ごとのテーブル命名（決定の要約）

| 層 | データセット | 命名 | 例 |
|---|---|---|---|
| Bronze（源泉宣言） | `bronze` | `<source>_<table>_raw` | `sap_cost_master_raw` |
| Silver | `intermediate` | `dim_*` / `fct_*` | `dim_department`, `fct_department_cost` |
| Gold | `outputs` | 業務名のみ（接頭辞なし） | `cost_management` |

- `outputs` のテーブル名は API エンドポイントと語彙を揃える（例: `/v1/cost-management` ↔ `outputs.cost_management`）。
- 洗浄層 `stg_<source>_<table>` の置き場所は `../02-architecture/bigquery-dataform-design-rules.md` の未決事項。
- 接頭辞の採否基準（冗長性の検査）と調査経緯は同書の付録。

### sources/（ソース宣言）の書き方

外部の生テーブルを `type: "declaration"` で宣言する層。ソースシステム名をファイル名・サブフォルダの両方に含める。

```sqlx
config {
  type: "declaration",
  schema: "bronze",
  name: "sap_cost_master_raw",
}
```

### outputs/（Gold）の書き方

```sqlx
config {
  type: "table",
  schema: "outputs",
  description: "コスト管理。1行＝部門×期間のコスト集計",
  assertions: {
    nonNull: ["department_id", "period"],
    uniqueKey: ["department_id", "period"],
  },
}
```

---

## 4. カラム命名

- 小文字 snake_case で統一（`order_id`, `created_at`, `total_amount`）。
- 主キー: `<entity>_id`（`order_id`）。外部キーも同名で揃え、結合を明快に。
- 日付/時刻: `*_date`（日付）/ `*_at`（タイムスタンプ）。例 `order_date`, `created_at`。
- 真偽値: `is_*` / `has_*`（`is_cancelled`）。
- 金額・数量はサフィックスで単位を示す（`amount_jpy`, `quantity` 等、チームで統一）。

---

## 5. その他の規約

| 対象 | 規約 |
|---|---|
| **schema（dataset）** | `config.schema` は `intermediate` / `outputs` 等のデータセット名と一致させる。環境（dev/stg/prod）の出し分けはデプロイ時に注入し、**名前にハードコードしない**（注入方式の正は `../02-architecture/bigquery-dataform-design-rules.md`・`../02-architecture/repository-strategy.md`） |
| **assertions** | ファイル名は対象テーブル基準で `assert_<table>_<rule>.sqlx`（例 `assert_cost_management_unique.sqlx`）。`intermediate` / `outputs` は document & test を必須 |
| **tags** | 実行単位を表す小文字 snake_case（`tag: ["daily", "cost"]`）。スケジュール/部分実行のフィルタに使う |
| **includes/** | 共通ヘルパのみ。`constants.js`（project/dataset を vars 経由）、`naming.js`、`assertions.js`。むやみに増やさない |

---

## 6. 既存大文字テーブルの移行（命名の対応例）

命名規約に合わせる際の名称対応の一例:

```
旧: Fact_COST_DETAIL
新: definitions/intermediate/fct_cost_detail.sqlx  →  テーブル fct_cost_detail
```

実際の移行は破壊的なので、順番・後方互換ビュー・契約変更としての段階適用・全エンティティの as-is→to-be 対応表は [`migration-plan.md`](migration-plan.md) が正本。本書（命名規約）としては「新規・改修分は最初から規約準拠」「リネームは契約変更扱い」だけ押さえる。

---

## 7. 公式リファレンスとの差分

Dataform 公式は `sources / intermediate / outputs` の段階構造、intermediate への固有プレフィックス、outputs の簡潔名を推奨する。本プロジェクトは段階構造を踏襲しつつ、**「フォルダ名 = データセット名（フラット配置）」と「Silver = `dim_`/`fct_`、Gold = 業務名のみ」をハウスルールとして優先**する（判断基準は `../02-architecture/bigquery-dataform-design-rules.md` 付録）。

- Structuring code in a repository（段階構造）
  https://cloud.google.com/dataform/docs/structure-repositories
- Best practices for repositories（全ファイル名は BigQuery テーブル命名規則に準拠／sources はソース名プレフィックス）
  https://cloud.google.com/dataform/docs/best-practices-repositories
- Best practices for the workflow lifecycle（dev/prod のソース宣言分離と命名一貫性）
  https://cloud.google.com/dataform/docs/managing-code-lifecycle
