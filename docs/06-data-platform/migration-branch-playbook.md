# Dataform 移行 ブランチ別 実施プレイブック

`migration-plan.md` §5「ブランチステップ（トランクベース）」を、**ブランチ 1 本ずつの実施手順**へ展開したもの。各ブランチで「何をやるか（実施事項）」「どのファイルを作る／変えるか（ファイル一覧と中身の雛形）」「どう検証するか」を 1 か所に集約する。

- 作成日: 2026-06-05
- 改訂日: 2026-06-11（命名を Silver=`dim_`/`fct_`・Gold=業務名へ追従。cost/order の集約先を `intermediate/` に変更、リポジトリ名を `common-data-model` に統一）
- 位置づけ: `migration-plan.md`（移行の順番・原則・名称マッピングの**正本**）と `../02-architecture/bigquery-dataform-design-rules.md`（層構造・テーブル命名の**正本**。詳細命名は `dataform-naming-convention.md`）の派生。本書は実装作業の手順書であり、正本と矛盾する場合は正本が優先する。
- ファイル中身の扱い: 掲載する `.sqlx` / `.yaml` / `.js` は**雛形（スケルトン）**。SQL ロジック本体は既存リポジトリ `ai-driven-comod-cdem` の各テーブル定義から移植する（本書では `-- 既存ロジックを移植` で示す）。`<…>` は Phase 0 の確認で確定する値。

---

## 0. 全体像（ブランチの並びと対応フェーズ）

`migration-plan.md` のロードマップ Step 1（手動・ローカル CLI）で構造集約を完結させる範囲。CI 自動デプロイ・WIF・Terraform（Step 2／3）、本筋の GCP ネイティブ移行（Step 4）は対象外。

| # | ブランチ | フェーズ | 目的（1 行） |
|---|---|---|---|
| 1 | `chore/repo-skeleton` | 1 | 単一リポジトリの器（環境非依存 `workflow_settings.yaml`・3 層・`includes/`・`ci.yml`）を作る |
| 2 | `feature/sap-coep-source` | 3 | `V_COEP` を `sources/sap/`（必要なら `intermediate/stg_coep`）へ |
| 3 | `feature/cost-intermediate` | 2＋3 | cost の 3 環境コピーを 1 本化し `intermediate/` へ（テーブル名は維持） |
| 4 | `feature/order-intermediate` | 2＋3 | order ドメインで同様 |
| 5 | `refactor/cost-rename` / `refactor/order-rename` | 4 | **消費者がいる場合のみ**。後方互換ビュー経由で小文字 snake_case へ |
| 6 | `chore/cleanup-legacy` | 5 | 雛形・残骸・rewrite スクリプト・旧 workflow を削除、`STRUCTURE.md`/`README.md` 更新 |

```
足す（骨格）─→ 畳む＋3層化（エンティティ単位）─→ （必要なら）リネーム ─→ 掃除
   #1              #2 / #3 / #4                      #5                #6
```

依存関係: #1 が全ブランチの前提。#2〜#4 は #1 後なら相互に独立で並行可。#5 は対象エンティティの #3／#4 後。#6 は最後。

---

## 1. 全ブランチ共通ルール（`migration-plan.md` §5 共通ルール）

`migration-plan.md` の「共通ルール」をブランチ作業の前提として再掲する。各ブランチの手順はこれを満たす。

- `main` を唯一の保護ブランチ。**PR 必須・CI green 必須・直 push 禁止・squash マージ**（1 PR = 1 revert 単位）。
- ブランチ命名: `feature/<entity>-<desc>` / `chore/<desc>` / `fix/<desc>` / `refactor/<desc>` / `hotfix/<desc>`。
- 各 PR の CI: `dataform compile` ＋ dry-run ＋ assertions。**dry-run は BigQuery に当たる**ため、CI ランナーに dev の最小 IAM と `bigquery.googleapis.com` 到達性が必要。
  - 補足（Step 1 期）: CI からの BigQuery 認証（WIF）は **Step 2 で必須化**される。それまでのゲートの実質は**ローカル CLI（ADC）run** で担保し、`ci.yml` は雛形として置きつつ compile までを確実に通す運用でよい。
- マージ後の流れ（Step 1）: `main` マージ後、開発者が**ローカル CLI** で `--default-database=<pj-dev>` を渡して dev に run → 検証。stg/prod は同一 commit に対し `--default-database` だけ差し替えて run（自動昇格は Step 2 の `promote.yml`）。
- feature 開発時の dev 書き込みは `--schema-suffix` 等で他開発者・prod と分離する。
- 認証: ローカルは ADC（`gcloud auth application-default login`）。**`.df-credentials.json` はコミットしない**。`environments/*.json` も非コミット。

各ブランチの典型コマンド（雛形）:

```bash
# 1) ブランチを切る
git switch -c <branch-name>

# 2) ローカル検証（dev へ）
dataform compile --default-database=<pj-dev>
dataform run     --default-database=<pj-dev> --dry-run        # まず dry-run
dataform run     --default-database=<pj-dev> --schema-suffix=<dev-handle>   # 実 run（自分専用接尾辞）

# 3) PR（CI green を待って squash マージ）
git push -u origin <branch-name>
gh pr create --fill
```

---

## 2. `chore/repo-skeleton`【Phase 1】

### 目的
到達点の器を用意する（既存ディレクトリと共存可）。GCP 側 Dataform リポジトリは**作らない**。

### 実施事項
- [ ] ルートに**環境非依存の単一 `workflow_settings.yaml`** を新設（`defaultProject`=dev 基準・`defaultLocation`・`dataformCoreVersion` を固定ピン・`vars` は共通のみ）。
- [ ] `definitions/{sources,intermediate,outputs}/` の 3 層ディレクトリを作成（中身は空、`.gitkeep`）。
- [ ] `includes/` に `constants.js` / `naming.js` / `assertions.js` を新設。
- [ ] `package.json`（`@dataform/core` をピン）を新設。
- [ ] `.github/workflows/ci.yml` を新設（compile＋dry-run＋assertions の雛形）。
- [ ] `.gitignore` に `.df-credentials.json` と `environments/*.json` を追加。
- [ ] ローカル CLI で `--default-database=<pj-dev>` を手で渡し、dev で**空コンパイル／run** が通ることを確認。

### 作成ファイル一覧
| ファイル | 役割 |
|---|---|
| `workflow_settings.yaml` | 環境非依存の中核設定。環境差は CLI で注入 |
| `definitions/sources/.gitkeep` | ① ソース宣言層（空） |
| `definitions/intermediate/.gitkeep` | ② 中間変換層（空） |
| `definitions/outputs/.gitkeep` | ③ 出力＝API 公開面（空） |
| `includes/constants.js` | project/dataset を vars 経由で参照する共通定数 |
| `includes/naming.js` | 命名規約マクロ |
| `includes/assertions.js` | 共通アサーション |
| `package.json` | `@dataform/core` のバージョンピン |
| `.github/workflows/ci.yml` | PR CI（compile＋dry-run＋assertions） |
| `.gitignore` | 資格情報・環境ファイルの除外 |

### ファイルの中身（雛形）

`workflow_settings.yaml`:
```yaml
# 環境非依存。dev/stg/prod の差は CLI の --default-database 等で注入する（名前にハードコードしない）。
dataformCoreVersion: "3.0.0"        # 固定ピン（build-once の前提。導入する @dataform/cli と一致させる）
defaultProject: "<pj-dev>"          # dev 基準。stg/prod は --default-database=<pj-stg|pj-prod> で上書き
defaultDataset: "outputs"           # 既定スキーマ。層別 schema は各 .sqlx の config.schema で指定
defaultLocation: "<asia-northeast1 など>"
defaultAssertionDataset: "dataform_assertions"
vars:
  # 共通の不変値のみ。環境差（project/dataset 名）はここに入れない
```

`includes/constants.js`:
```js
// 環境差は CLI の --default-database / --vars で注入する。ここには全環境共通の不変値のみを置く。
module.exports = {
  // ソースシステムの生データセット名（Phase 0 で確定）
  RAW_SAP: "<raw_sap>",
  // 出力・中間の schema（環境では変えない。project だけ --default-database で差し替え）
  SCHEMA_OUTPUTS: "outputs",
  SCHEMA_INTERMEDIATE: "intermediate",
};
```

`includes/naming.js`:
```js
// 命名規約（小文字 snake_case）を補助するマクロ置き場。最初は最小で可。
module.exports = {
  // 例: 出力テーブルの description 定型などをここに集約していく
};
```

`includes/assertions.js`:
```js
// 共通アサーションのヘルパ。各 outputs/intermediate から呼び出す。
module.exports = {
  // 例: 共通の rowConditions テンプレート等をここに集約していく
};
```

`package.json`:
```json
{
  "name": "dataform",
  "dependencies": {
    "@dataform/core": "3.0.0"
  }
}
```

`.github/workflows/ci.yml`（雛形。WIF 認証の本格化は Step 2）:
```yaml
name: dataform-ci
on:
  pull_request:
    branches: [main]
jobs:
  compile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20" }
      - run: npm ci
      - run: npx dataform compile
      # 以下 dry-run / assertions は BigQuery 到達が必要。
      # Step 2 で WIF(OIDC)＋dev 最小 IAM を入れてから有効化する。
      # - run: npx dataform run --dry-run --default-database=${{ vars.PJ_DEV }}
```

`.gitignore`（追記）:
```gitignore
.df-credentials.json
environments/*.json
node_modules/
```

### 検証 / 完了条件
- `dataform compile` がローカルの最小 IAM で通る（**compile は BigQuery に当たるためオフライン完結不可**）。
- `dataform run --dry-run --default-database=<pj-dev>` が空構成で成功。
- 既存ディレクトリと共存できている（既存を壊していない）。

> 備考: 環境差を持ち込んだ `environments/*.json` は**コミットしない**。Step 1 では開発者が `--default-database`（必要に応じ `--schema-suffix` / `--vars`）を手で渡す。

---

## 3. `feature/sap-coep-source`【Phase 3】

### 目的
`V_COEP` を 3 層構造の **sources 層**へ正しく配置する（生 SAP は declaration）。リネームはまだしない（名前は維持／層だけ確定）。

### 実施事項
- [ ] Phase 0 の確認結果に基づき、`V_COEP` の中身が「生テーブルの宣言（declaration）」か「変換あり」かを判定。
- [ ] 生 SAP テーブルを `sources/sap/` に `type: "declaration"` で宣言。
- [ ] 変換ロジックがある場合のみ、`intermediate/stg_coep.sqlx` に変換を切り出す（`stg_` プレフィックス必須）。
- [ ] ローカル CLI で dev に run → 既存と同等の結果を確認。

### 作成ファイル一覧
| ファイル | 条件 | 役割 |
|---|---|---|
| `definitions/sources/sap/coep.sqlx` | 常時 | 生 SAP COEP の declaration |
| `definitions/intermediate/stg_coep.sqlx` | 変換ありの場合のみ | `V_COEP` の変換ロジック（API 非公開） |

### ファイルの中身（雛形）

`definitions/sources/sap/coep.sqlx`（declaration。ソースシステム名 `sap` をプレフィックス＝ディレクトリで表現）:
```sqlx
config {
  type: "declaration",
  schema: "<raw_sap>",     -- 生データセット（Phase 0 で確定）
  name: "coep",            -- SAP CO 実績明細
}
```

`definitions/intermediate/stg_coep.sqlx`（変換がある場合のみ）:
```sqlx
config {
  type: "view",            -- もしくは "table"。元 V_COEP がビューなら view 相当
  schema: "intermediate",
  description: "SAP CO 実績明細の整形（API 非公開の中間層）",
}

-- 既存 V_COEP の SELECT ロジックを移植。生テーブルは ref ではなく declaration を参照
SELECT
  -- 既存ロジックを移植
FROM ${ref("coep")}
```

### 検証 / 完了条件
- dev で run し、`V_COEP` 相当の出力が再生成される（行数・主要列の突き合わせ）。
- 層の判断（`sources` だけで足りるか `intermediate` が要るか）が確定し、ファイル配置に反映されている。

> 注: `coep` という最終名にするか `stg_coep` を併設するかは `migration-plan.md` §2 名称マッピングの「層は要確認」に従う。本ブランチでは**層の確定**までを行い、公開面リネームは Phase 4 の範囲。

---

## 4. `feature/cost-intermediate`【Phase 2＋3】

### 目的
cost ドメインの **3 環境コピー（`*` / `*-stg` / `*-prd`）を 1 本に畳み**、`intermediate/` へ集約する。**テーブル名は現状維持**（大文字のまま）。`dataform-rewrite.sh`（`sed` 置換）と REST `writeFile` への依存をこのドメインから外す。

> 消費者（API `repository.py` / BI）が**いない**場合は、Phase 4 を待たず**このブランチでリネーム**してよい（最も簡素）。いる場合は名前維持のまま畳み、リネームは #5 で。

### 実施事項
- [ ] `Dim_COST_MASTER{,-stg,-prd}` の 3 コピーを diff し、差が「project/dataset 名だけ」か「SQL ドリフト」かを確認（Phase 0 の結論を反映。ドリフトなら **prod を真**として正す）。
- [ ] `Fact_COST_DETAIL{,-stg,-prd}` も同様に diff。
- [ ] 環境差を**デプロイ時の `--default-database=<pj-env>`**で吸収する前提で、SQL から環境ハードコードを除去。
- [ ] master → `intermediate/dim_cost_master.sqlx`、detail → `intermediate/fct_cost_detail.sqlx` に集約。
- [ ] `config` に `description` と assertions（`nonNull` / `uniqueKey`）を付与。
- [ ] 同一 commit に対し `--default-database` を dev→stg→prod と差し替えて run し、結果一致を確認（build-once の素振り）。

### 名称マッピング（`migration-plan.md` §2）
| as-is（3 コピー） | to-be ファイル | 維持するテーブル名（消費者あり） | リネーム後（消費者なし） |
|---|---|---|---|
| `Dim_COST_MASTER{,-stg,-prd}` | `intermediate/dim_cost_master.sqlx` | `Dim_COST_MASTER` | `dim_cost_master` |
| `Fact_COST_DETAIL{,-stg,-prd}` | `intermediate/fct_cost_detail.sqlx` | `Fact_COST_DETAIL` | `fct_cost_detail` |

### ファイルの中身（雛形）

`definitions/intermediate/dim_cost_master.sqlx`（**名前維持**パターン。消費者ありで Phase 4 まで旧名を保つ）:
```sqlx
config {
  type: "table",
  schema: "intermediate",
  name: "Dim_COST_MASTER",   -- Phase 2+3 は名前維持。消費者なしなら "dim_cost_master" にしてこの行を削除可
  description: "原価マスタ。1行＝原価コード1件",
  assertions: {
    nonNull: ["<pk_col>"],
    uniqueKey: ["<pk_col>"],
  },
}

-- 3 コピー(*/-stg/-prd)で同一だった SELECT を 1 本化して移植。
-- project/dataset のハードコードは置かない（環境は --default-database で注入）。
SELECT
  -- 既存ロジックを移植
FROM ${ref("<source_or_intermediate>")}
```

`definitions/intermediate/fct_cost_detail.sqlx`:
```sqlx
config {
  type: "table",
  schema: "intermediate",
  name: "Fact_COST_DETAIL",  -- 同上（消費者なしなら "fct_cost_detail"）
  description: "原価明細。1行＝原価トランザクション1件",
  assertions: {
    nonNull: ["<pk_col>", "<fk_col>"],
    uniqueKey: ["<pk_col>"],
  },
}

-- 既存ロジックを移植
SELECT
  -- 既存ロジックを移植
FROM ${ref("coep")}      -- 例: #2 で宣言した SAP ソース／中間を参照
```

（任意）スタンドアロン assertion を切る場合 `definitions/intermediate/assert_cost_detail_unique.sqlx`:
```sqlx
config { type: "assertion" }
SELECT <pk_col>, COUNT(*) c
FROM ${ref("Fact_COST_DETAIL")}   -- 維持中の名前 / リネーム後は fct_cost_detail
GROUP BY 1 HAVING c > 1
```

### 検証 / 完了条件
- dev に run し、畳む前と**同一テーブル**が再生成される（行数・スキーマ・サンプル値）。
- `--default-database` だけを stg→prod に差し替えた run で結果一致（build-once）。
- BigQuery 上のテーブル名を変えていない（消費者ありの場合）＝下流に無影響。

> 非破壊性: 名前を維持する限り下流（API `repository.py` / BI）に影響しない。Phase 2 と 3 はエンティティ単位で 1 ブランチにまとめると効率的、という方針どおり本ブランチで両方を済ませる。

---

## 5. `feature/order-intermediate`【Phase 2＋3】

### 目的
order ドメインで #4 と同じことを行う。

### 実施事項
- [ ] `Dim_ORDER_MASTER{,-stg,-prd}` / `Fact_ORDER_DETAIL{,-stg,-prd}` を diff（環境差のみか／ドリフトか）。
- [ ] master → `intermediate/dim_order_master.sqlx`、detail → `intermediate/fct_order_detail.sqlx` に集約（名前維持）。
- [ ] `description` ＋ assertions 付与。
- [ ] build-once の素振り（dev→stg→prod で `--default-database` 差し替え一致）。

### 名称マッピング
| as-is（3 コピー） | to-be ファイル | 維持するテーブル名 | リネーム後 |
|---|---|---|---|
| `Dim_ORDER_MASTER{,-stg,-prd}` | `intermediate/dim_order_master.sqlx` | `Dim_ORDER_MASTER` | `dim_order_master` |
| `Fact_ORDER_DETAIL{,-stg,-prd}` | `intermediate/fct_order_detail.sqlx` | `Fact_ORDER_DETAIL` | `fct_order_detail` |

### ファイルの中身（雛形）

`definitions/intermediate/dim_order_master.sqlx`:
```sqlx
config {
  type: "table",
  schema: "intermediate",
  name: "Dim_ORDER_MASTER",   -- 消費者なしなら "dim_order_master"
  description: "注文マスタ",
  assertions: {
    nonNull: ["order_id"],
    uniqueKey: ["order_id"],
  },
}

-- 既存ロジックを移植
SELECT
  -- 既存ロジックを移植
FROM ${ref("<source_or_intermediate>")}
```

`definitions/intermediate/fct_order_detail.sqlx`:
```sqlx
config {
  type: "table",
  schema: "intermediate",
  name: "Fact_ORDER_DETAIL",  -- 消費者なしなら "fct_order_detail"
  description: "注文明細。1行＝注文1明細",
  assertions: {
    nonNull: ["order_id", "line_no"],
    uniqueKey: ["order_id", "line_no"],
  },
}

-- 既存ロジックを移植
SELECT
  -- 既存ロジックを移植
FROM ${ref("<source_or_intermediate>")}
```

### 検証 / 完了条件
- #4 と同じ基準（畳む前と同一テーブル再生成、build-once 素振り、名前維持で下流無影響）。

---

## 6. `refactor/cost-rename` / `refactor/order-rename`【Phase 4】

### 目的
小文字 snake_case 化（`Dim_`/`Fact_` 撤去）。**消費者がいる場合のみ**のブランチ。消費者なしなら #4／#5 で既にリネーム済みなので本ブランチは不要。

### 実施事項（エンティティ単位で 1 ブランチずつ）
- [ ] 新名（例 `dim_cost_master`）の `type: "table"` を作成（中身は既存のまま `name` を新名に）。
- [ ] 旧名（例 `Dim_COST_MASTER`）を **`type: "view"` で残し新名を指す**後方互換ビューを別ファイルで定義。
- [ ] 旧名・新名の両方を参照テストで確認。
- [ ] 下流（API `repository.py` / BI）を新名へ移行。`data-contracts`（採用時）の契約・API 側スキーマと突き合わせてから適用。
- [ ] 下流移行完了後、後方互換ビュー（旧名）を撤去 → 旧名の DROP は #6 で。

### ファイルの中身（雛形）

新名テーブル `definitions/intermediate/dim_cost_master.sqlx`（`name` を新名へ。`config.name` 省略でファイル名＝テーブル名でも可）:
```sqlx
config {
  type: "table",
  schema: "intermediate",
  description: "原価マスタ（Silver ディメンション）",
  assertions: { nonNull: ["<pk_col>"], uniqueKey: ["<pk_col>"] },
}
-- 既存ロジックを移植（#4 の中身をそのまま）
SELECT /* … */ FROM ${ref("<source_or_intermediate>")}
```

後方互換ビュー `definitions/intermediate/legacy_dim_cost_master.sqlx`（**一時的**。旧名で新名を指す）:
```sqlx
config {
  type: "view",
  schema: "intermediate",
  name: "Dim_COST_MASTER",     -- 旧名を view として一時的に温存
  description: "後方互換: 旧名 Dim_COST_MASTER → 新 dim_cost_master。下流移行後に撤去",
}
SELECT * FROM ${ref("dim_cost_master")}
```

> order 側も同型（`dim_order_master` / `fct_order_detail` を新名、`Dim_ORDER_MASTER` / `Fact_ORDER_DETAIL` を後方互換 view）。

### 検証 / 完了条件
- 新名テーブルと旧名ビューの双方が dev で生成・参照可能。
- 下流が新名参照に切り替わったことを確認 → 旧名ビュー撤去。
- 破壊的変更を契約変更として扱い、段階適用できている（1 エンティティずつ）。

---

## 7. `chore/cleanup-legacy`【Phase 5】

### 目的
残骸の除去と仕上げ。単一リポジトリ＝単一 Dataform repo・3 層・命名規約・build-once が満たされた状態にする。

### 実施事項
- [ ] 雛形・残骸ディレクトリを削除: `empty/` `github-actions/` `develop/` `staging/`。
- [ ] 各ディレクトリの `dataform-rewrite.sh`（`sed` 置換）と個別 `workflows_dataform_write.yml` を削除。
- [ ] ルートの旧 workflow を削除。
- [ ] 旧テーブル別×環境別ディレクトリ（`Dim_*` / `Fact_* ` / `V_*` の `*`/`*-stg`/`*-prd`）が #2〜#6 移行で空になっていることを確認して削除。
- [ ] `STRUCTURE.md` / `README.md` を新構造に更新。
- [ ] 契約移行が完了した旧大文字テーブルが BigQuery に残っていれば **DROP**（#6 のリネーム完了後）。

### 削除対象一覧（as-is → 削除）
| 対象 | 理由 |
|---|---|
| `empty/` | 空 Dataform repo の雛形（暫定 CLI 路線で不要） |
| `github-actions/` | CI＋rewrite 同梱の雛形（不要） |
| `develop/` `staging/` | branch-per-env の名残（README のみ） |
| `*/dataform-rewrite.sh` | `sed` 置換は環境差 CLI 注入で不要 |
| `*/workflows_dataform_write.yml` | REST `writeFile` 経路は廃止 |
| ルート旧 workflow | 新 `ci.yml`（＋Step 2 の deploy/promote）に置換 |
| 旧 `Dim_*` / `Fact_*` / `V_*` の env コピー dir | #2〜#5 で `definitions/` 配下へ移設済み |

### 更新ファイルの中身（雛形）

`STRUCTURE.md`（新構造へ）:
```markdown
# リポジトリ構成

common-data-model/
├── definitions/
│   ├── sources/<source_system>/      # declaration（生テーブル）
│   ├── intermediate/{dim_,fct_}*.sqlx # Silver（API 非公開）
│   └── outputs/<entity>.sqlx          # Gold＝API 公開面（業務名・フラット）
├── includes/{constants,naming,assertions}.js
├── workflow_settings.yaml            # 環境非依存。環境差は CLI --default-database で注入
├── package.json
└── .github/workflows/ci.yml          # PR: compile + dry-run + assertions
```

`README.md`（運用節を更新。要点のみ）:
```markdown
## デプロイ（暫定: CLI 路線）
- 環境差は `--default-database=<pj-dev|pj-stg|pj-prod>` で注入。コードは全環境同一（build-once）。
- ローカルは ADC で run（`.df-credentials.json` は非コミット）。
- 自動デプロイ・stg/prod 昇格・WIF は Step 2（`deploy-dev.yml` / `promote.yml`）。
- 本筋は Terraform 管理の release / workflow config へ移行予定（正本: `02-architecture/bigquery-dataform-design-rules.md`）。
```

### 検証 / 完了条件（`migration-plan.md` §8 完了基準と一致）
- [ ] 単一 Dataform リポジトリ＝単一 Git リポジトリ。
- [ ] `definitions/` が sources/intermediate/outputs の 3 層。
- [ ] 命名規約（小文字 snake_case。Silver=`dim_`/`fct_`、Gold=業務名のみ）を満たす（#5 完了が前提）。
- [ ] `workflow_settings.yaml` は環境非依存。環境差は `--default-database`（必要に応じ `--schema-suffix`/`--vars`）。`environments/*.json` は非コミット。
- [ ] 同一 commit から dev/stg/prod を再生成できる（build-once の素振り確認）。
- [ ] `empty/`・`github-actions/`・`develop/`・`staging/`・`dataform-rewrite.sh`・個別 workflow を削除済み。

---

## 8. ブランチをまたぐ前提（Phase 0 で先に潰す）

各ブランチの diff／層判定はすべて **Phase 0（確認・準備）** の結論に依存する。`chore/repo-skeleton` に着手する前に以下を済ませておく（`migration-plan.md` §8 実行チェックリスト Phase 0）。

- [ ] `*` / `*-stg` / `*-prd` の `workflow_settings.yaml` と `.sqlx` を diff（環境差は project だけか／SQL ドリフトはないか。ドリフトなら **prod を真**）。
- [ ] 下流消費者（API `repository.py` / BI）の有無 → **リネーム方式（#4/#5 で即リネームか、後方互換ビュー経由か）が分岐**。
- [ ] `V_COEP` の中身（declaration か変換ありか）→ #2 の層が決まる。
- [ ] dev の最小 IAM（compile の dry-run／run に必要な BigQuery 権限）。
- [ ] proxy-only egress 下の `HTTPS_PROXY`・npm registry・`bigquery.googleapis.com` 到達性。

---

## 9. ロールバック・安全策（全ブランチ共通）

`migration-plan.md` §7 と同じ。

- **squash** により「1 PR = 1 revert」。問題があれば該当 PR を revert。
- **build-once**: 同一 commit を同一 `dataformCoreVersion` でコンパイルした結果を各環境に run。前の安定 commit / tag を再 run すれば即復旧。
- **fix-forward**: BigQuery テーブルや環境設定を手で直さない。直すのは `main`、dev で再現確認してから昇格。
- 不具合は**再発防止の assertion / テスト**を必ず追加し、dev のゲートで二度と素通りさせない。

---

## 参考
- 移行の順番・原則・名称マッピング（正本）: [`migration-plan.md`](migration-plan.md)
- 命名・層構造（正本）: [`dataform-naming-convention.md`](dataform-naming-convention.md)
- 上位設計（分割・昇格/修正フロー・ブランチ戦略）: [`../02-architecture/repository-strategy.md`](../02-architecture/repository-strategy.md)
- 本筋（GCP ネイティブ方式）の基礎検討記録: [`dataform-operating-model.md`](dataform-operating-model.md)
