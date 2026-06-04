# Dataform リポジトリ構成・命名規約

`dataform` リポジトリ（→ BigQuery）のフォルダ体系とファイル/テーブル命名規約をまとめたリファレンス。Dataform 公式の推奨に素直に従う方針で確定。

- 作成日: 2026-05-31
- ステータス: 確定（公式標準 `sources / intermediate / outputs` 準拠）
- 関連: `repository-strategy.md`（リポジトリ分割・昇格フロー）

---

## 1. 大原則

| 原則 | 内容 |
|---|---|
| **小文字 snake_case** | テーブル名・カラム名・ファイル名すべて。大文字は使わない（`ORDER_DETAIL` ✕ → `order_detail` ○） |
| **ファイル名 = テーブル名** | `definitions/.../order_detail.sqlx` → BigQuery テーブル `order_detail`。一致させる |
| **ファイル名はサブディレクトリ構造を反映** | Dataform 公式の明示要件。フォルダで役割が分かるようにする |
| **BigQuery 命名規則に準拠** | 全ファイル名が BigQuery のテーブル命名規則に従う必要がある（Dataform の制約） |
| **フォルダ = ドメイン分け、環境分けには使わない** | 環境差（dev/stg/prod）はデプロイ時に注入し、フォルダやブランチで環境を分けない（注入方式＝CLI コンパイラオプションは `repository-strategy.md`・`migration-plan.md` が正） |

> 注: `fct_` / `dim_` プレフィックス（dbt 流のディメンショナルモデリング）は **採用しない**。Dataform 公式は `outputs` のファイル名を「簡潔に」とのみ規定しており、層プレフィックスを付けるのは中間層（`stg_`）のみとする。

---

## 2. ディレクトリ構成

```
dataform/
├── definitions/
│   ├── sources/                      # ① ソース宣言（declaration）＋軽い変換
│   │   └── <source_system>/
│   │       ├── <source_system>.sqlx          # declaration（生テーブル）
│   │       └── <source_system>_filtered.sqlx # フィルタ等の軽い変換（任意）
│   ├── intermediate/                 # ② 結合・本格的な変換（API非公開）
│   │   └── stg_<topic>.sqlx
│   └── outputs/                      # ③ 出力テーブル＝API公開面（エンティティ単位）
│       ├── order/
│       │   └── order_detail.sqlx
│       ├── requisition/
│       │   └── requisition.sqlx
│       ├── invoice/
│       │   └── invoice.sqlx
│       └── man_hour/
│           └── man_hour.sqlx
├── includes/                         # 共通 JS/SQLX ヘルパ（増やさない）
│   ├── constants.js                  #   project/dataset を vars 経由で参照
│   ├── naming.js                     #   命名規約マクロ
│   └── assertions.js                 #   共通アサーション
├── workflow_settings.yaml            # 環境非依存: defaultProject/Dataset/Location, dataformCoreVersion, vars
├── package.json
│   # ★環境差（dev/stg/prod）はデプロイ時の CLI コンパイラオプション（--default-database 等）で注入。
│   #   environments/*.json は使うとしてもローカル用で「非コミット」（CLI 路線。repository-strategy.md・migration-plan.md と整合）。
└── .github/workflows/
```

`definitions/` 直下を **ワークフローの段階（sources → intermediate → outputs）** で 3 分割するのが Dataform 公式の標準構造。各段階の中をビジネスドメイン（`order`, `invoice` …）のサブディレクトリで分ける。テーブルが増えてもフォルダ構成は変えず、ドメインディレクトリを足していく。

---

## 3. 層ごとの命名規約

### ① sources/（ソース宣言）

外部の生テーブルを `type: "declaration"` で宣言する層。+ 軽いフィルタ程度の変換まで。

- **ソースシステム名をプレフィックス**にする（公式要件）。
- ファイル名でどの外部ソース由来か分かるようにする。

| 種別 | 命名例 |
|---|---|
| declaration | `erp.sqlx` / `erp_orders.sqlx` |
| 軽い変換 | `erp_orders_filtered.sqlx` |

```sqlx
config {
  type: "declaration",
  schema: "raw_erp",
  name: "orders",
}
```

### ② intermediate/（中間変換）

複数ソースの結合や本格的な変換ロジック。**BI ツール・API には公開しない**前提。

- **中間ディレクトリ専用の固有プレフィックス `stg_` を付ける**（公式要件: 中間ファイルだと一目で分かるようにする）。
- すべて document & test（assertions）する。

| 命名例 |
|---|
| `stg_orders_joined.sqlx` |
| `stg_order_line_enriched.sqlx` |

### ③ outputs/（出力テーブル）

ビジネス/分析目的の最終テーブル。本プロジェクトでは **API の公開面**に対応し、エンティティ単位のサブディレクトリで管理。

- **ファイル名は簡潔に**（公式要件）。層プレフィックスは付けない。
- ディレクトリ（ドメイン）＋簡潔なテーブル名で表現する。

| ドメイン（ディレクトリ） | ファイル＝テーブル名 |
|---|---|
| `outputs/order/` | `order_detail.sqlx`, `order.sqlx` |
| `outputs/invoice/` | `invoice.sqlx`, `invoice_line.sqlx` |
| `outputs/man_hour/` | `man_hour.sqlx` |

```sqlx
config {
  type: "table",
  schema: "outputs",        -- 環境差は vars で吸収
  description: "注文明細。1行＝注文1明細",
  assertions: {
    nonNull: ["order_id", "line_no"],
    uniqueKey: ["order_id", "line_no"],
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
| **schema（dataset）** | `config.schema` は `sources` 系 / `intermediate` / `outputs` 等で分ける。環境（dev/stg/prod）の出し分けはデプロイ時の CLI コンパイラオプション（`--default-database` 等）で注入し、**名前にハードコードしない**（注入方式の正は `repository-strategy.md`・`migration-plan.md`） |
| **assertions** | ファイル名は対象テーブル基準で `assert_<table>_<rule>.sqlx`（例 `assert_order_detail_unique.sqlx`）。`intermediate` / `outputs` は document & test を必須 |
| **tags** | 実行単位を表す小文字 snake_case（`tag: ["daily", "order"]`）。スケジュール/部分実行のフィルタに使う |
| **includes/** | 共通ヘルパのみ。`constants.js`（project/dataset を vars 経由）、`naming.js`、`assertions.js`。むやみに増やさない |

---

## 6. 既存大文字テーブルの移行（命名の対応例）

命名規約に合わせる際の名称対応の一例:

```
旧: ORDER_DETAIL
新: definitions/outputs/order/order_detail.sqlx  →  テーブル order_detail
```

実際の移行は破壊的なので、順番・後方互換ビュー・契約変更としての段階適用・全エンティティの as-is→to-be 対応表は [`migration-plan.md`](migration-plan.md) が正本。本書（命名規約）としては「新規・改修分は最初から小文字 snake_case」「リネームは契約変更扱い」だけ押さえる。

---

## 7. 公式リファレンス

- Structuring code in a repository（`sources`/`intermediate`/`outputs` の段階構造、ファイル名はサブディレクトリ構造を反映）
  https://cloud.google.com/dataform/docs/structure-repositories
- Best practices for repositories（全ファイル名は BigQuery テーブル命名規則に準拠／sources はソース名プレフィックス／intermediate は固有プレフィックス／outputs は簡潔に）
  https://cloud.google.com/dataform/docs/best-practices-repositories
- Best practices for the workflow lifecycle（dev/prod のソース宣言分離と命名一貫性）
  https://cloud.google.com/dataform/docs/managing-code-lifecycle
