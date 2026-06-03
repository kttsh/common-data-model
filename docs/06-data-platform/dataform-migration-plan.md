# Dataform 既存リポジトリ改造プラン（移行手順・ブランチステップ）

`ai-driven-comod-cdem`（全ブランチを統合した `main`）を、`dataform-naming-convention.md` と `dataform-operating-model.md` で定めた **あるべき姿**へ移行する手順。破壊を避ける**順番**と、トランクベースの**ブランチステップ**をまとめる。

- 作成日: 2026-06-03
- ステータス: ドラフト（実行手順）
- 関連: `dataform-operating-model.md`（到達点の運用設計）、`dataform-naming-convention.md`（命名・層）、`repository-strategy.md`（案 B 全体）
- 前提: GCP ネイティブ（単一リポジトリ＋release configuration のプロジェクトオーバーライド、strict act-as カスタム SA）

---

## 1. 現状（as-is）の要約と問題

全ブランチを統合した結果、**テーブル別×環境別ディレクトリ**が乱立し、各ディレクトリが独立した `workflow_settings.yaml` を持つ（＝実質それぞれ別 Dataform リポジトリ）。

```
ai-driven-comod-cdem/
├── Dim_COST_MASTER/        Dim_COST_MASTER-stg/   Dim_COST_MASTER-prd/
├── Dim_ORDER_MASTER/       …-stg/                 …-prd/
├── Fact_COST_DETAIL/       …-stg/                 …-prd/
├── Fact_ORDER_DETAIL/      …-stg/                 …-prd/
├── V_COEP/                 …-stg/                 …-prd/
├── empty/                  # 空 Dataform repo の雛形
├── github-actions/         # CI＋rewrite 同梱の雛形
├── develop/  staging/      # README のみ（branch-per-env の名残）
└── .github/ .vscode/ README.md STRUCTURE.md .gitignore
```

| as-is の状態 | 抵触している原則 | 出典 |
|---|---|---|
| 大文字 `Dim_COST_MASTER` / `V_COEP` | 小文字 snake_case | 命名 §1 |
| `Dim_` / `Fact_` / `V_` プレフィックス | dim/fct プレフィックス不採用、`outputs` は簡潔名 | 命名 §1・§3③ |
| `-stg` / `-prd` を物理コピーで分離 | 環境差はフォルダ/ブランチで分けず override で吸収 | 命名 §1、運用設計 §3 |
| 各テーブルが個別 `workflow_settings.yaml`＝実質別 repo | 1 Git = 1 Dataform repo、`outputs/<entity>/` で増やす | 運用設計 §2・§5 |
| `outputs` しか無い（`sources`/`intermediate` 無し） | `definitions/` を 3 層に分割 | 命名 §2 |
| `dataform-rewrite.sh`（`sed` 置換）＋ REST `writeFile` | release configuration のオーバーライドへ | 運用設計 §3・§6 |
| SA JSON キー＋既定サービスエージェント | カスタム SA ＋ actAs（strict act-as 必須） | 運用設計 §4 |
| `empty/`・`github-actions/`・`develop/`・`staging/` | 雛形・残骸。ネイティブでは「`outputs/<entity>/` を1ファイル足す」運用 | 運用設計 §5 |

---

## 2. あるべき姿（to-be）— 要約

単一リポジトリ・3 層構造・小文字 snake_case・環境差は release configuration・strict act-as カスタム SA。完成形ツリーは `dataform-operating-model.md` §5 を参照。

### 名称マッピング

| as-is | 種別 | to-be | 配置 |
|---|---|---|---|
| `Dim_COST_MASTER` | master | `cost_master` | `outputs/cost/cost_master.sqlx` |
| `Fact_COST_DETAIL` | detail | `cost_detail` | `outputs/cost/cost_detail.sqlx` |
| `Dim_ORDER_MASTER` | master | `order_master` | `outputs/order/order_master.sqlx` |
| `Fact_ORDER_DETAIL` | detail | `order_detail` | `outputs/order/order_detail.sqlx` |
| `V_COEP` | SAP CO ビュー | `coep`（＋必要なら `stg_coep`） | `sources/sap/coep.sqlx`（層は要確認） |

---

## 3. 移行の大原則（順番の根拠）

1. **構造の集約（低リスク・先）と命名リネーム（破壊的・後）を分離する。**
2. **環境コピーの集約は、BigQuery テーブル名を変えなければ下流（API `repository.py` / BI）に無傷。** だから先に畳んで、最も痛い「3 重コピー・ドリフト・cherry-pick 地獄」を消す。
3. **リネームは契約変更として扱う。** 大文字＋`Dim_`/`Fact_` → 小文字 snake_case はテーブル名が変わるので、消費者がいる場合は後方互換ビューを置き、**エンティティ単位**で段階適用する（命名 §6）。
4. **strict act-as は時限（既存 repo は 2026-04-29〜07-31 に段階適用）。** Phase 1 で並行して必須対応する。

---

## 4. フェーズ（順番）

### Phase 0 — 確認・準備（破壊なし）
- 目的: 事故要因の洗い出し。
- 作業: ① `*` / `*-stg` / `*-prd` の `workflow_settings.yaml` と `.sqlx` を **diff** し、環境差が「project/dataset 名だけ」か「SQL ロジックも違う（ドリフト）」かを判定（ドリフトしていれば **prod を真**として正す）。② 下流消費者（API/BI）の有無を確認。③ `V_COEP` の中身を確認し `sources` か `intermediate` かを決める。④ 組織ポリシー `dataform.restrictGitRemotes` の許可リスト、カスタム SA 準備を確認。
- 検証: インベントリ表が揃い、リネーム方式（§6 の分岐）が決まる。

### Phase 1 — 単一リポジトリ骨格 ＋ ネイティブデプロイ基盤
- 目的: 到達点の器を用意する（既存ディレクトリと共存可）。
- 作業: ルートに単一 `workflow_settings.yaml`（環境非依存）・`definitions/{sources,intermediate,outputs}/`・`includes/`・`.github/workflows/ci.yml` を新設。GCP 側で Dataform リポジトリ 1 つを GitHub に接続、**カスタム SA ＋ actAs**、`dev`/`stg`/`prod` の release configuration（プロジェクトオーバーライド）と簡易 workflow configuration をスケルトンとして作成（Terraform 推奨）。
- 検証: dev release/workflow が空コンパイルで通る。strict act-as でカスタム SA 実行が成功する。

### Phase 2 — 環境コピーを畳む（名前は維持）
- 目的: per-env コピーの解消。
- 作業: `*` / `*-stg` / `*-prd` の 3 コピーを 1 本に集約。環境差は release configuration の **プロジェクトオーバーライド**に移し、`dataform-rewrite.sh` を廃止。**テーブル名は現状維持（大文字のまま）**。
- 検証: dev へ run し、畳む前と同一テーブルが再生成される。同一 commit を stg→prod に固定して run し一致を確認。
- 非破壊性: BigQuery 上の名前を変えないため下流に影響しない。

### Phase 3 — 3 層構造へ再配置（名前は維持）
- 目的: `definitions/` の標準化。
- 作業: `V_COEP` を `sources/sap/`（または `intermediate/stg_coep`）へ。master/detail を `outputs/<domain>/`（`cost/`・`order/`）へ移動。**まだリネームしない**。
- 検証: 各移動を dev で run → 昇格。
- 備考: Phase 2 と 3 はエンティティ単位で 1 ブランチにまとめると効率的（§5 参照）。

### Phase 4 — 命名規約への移行（破壊的・エンティティ単位）
- 目的: 小文字 snake_case 化。
- 作業: §6 の分岐に従う（消費者なしなら Phase 3 と同時、ありなら後方互換ビュー経由で段階移行）。
- 検証: 旧名・新名の両方を参照テストで確認してから旧名を撤去。

### Phase 5 — クリーンアップ ＋ strict act-as 完了確認
- 目的: 残骸の除去と仕上げ。
- 作業: `empty/`・`github-actions/`・`develop/`・`staging/`、各ディレクトリの `dataform-rewrite.sh` と個別 `workflows_dataform_write.yml`、ルートの旧 workflow を削除。`STRUCTURE.md`/`README.md` を新構造に更新。旧大文字テーブルが残っていれば契約移行完了後に DROP。
- 検証: strict act-as の段階適用期限までに全 release/workflow がカスタム SA で稼働していること。

---

## 5. ブランチステップ（トランクベース）

### 共通ルール
- `main` を唯一の保護ブランチ。PR 必須・CI green 必須・直 push 禁止・**squash マージ**（1 PR = 1 revert 単位）。
- 命名: `feature/<entity>-<desc>` / `chore/<desc>` / `fix/<desc>` / `refactor/<desc>` / `hotfix/<desc>`。
- 各 PR の CI は `dataform compile` ＋ dry-run ＋ assertions。
- マージ後の流れ: `main` → dev release/workflow が run（pj-dev）→ 検証 → **同一 commit を stg→prod に固定**して run（承認ゲートは当面 §運用設計7 の手動前進）。
- feature 開発時の dev 書き込みは **workspace compilation override の schema suffix** で他開発者・prod と分離。

### 順序付きブランチ

| # | ブランチ | フェーズ | 内容 |
|---|---|---|---|
| 1 | `chore/repo-skeleton` | 1 | 単一 `workflow_settings.yaml`・`definitions/{sources,intermediate,outputs}`・`includes/`・`ci.yml` を新設（既存 dir と共存） |
| 2 | `chore/native-deploy-setup` | 1 | Git 接続・カスタム SA＋actAs・`dev`/`stg`/`prod` release configuration（project override）・簡易 workflow configuration・CI の OIDC/WIF（Terraform 化） |
| 3 | `feature/sap-coep-source` | 3 | `V_COEP` を `sources/sap/coep.sqlx`（または `intermediate/stg_coep`）へ。生 SAP は declaration |
| 4 | `feature/cost-outputs` | 2＋3 | `Dim_COST_MASTER{,-stg,-prd}`＋`Fact_COST_DETAIL{,-stg,-prd}` を `outputs/cost/{cost_master,cost_detail}.sqlx` に集約。名前維持（消費者なしならここでリネーム） |
| 5 | `feature/order-outputs` | 2＋3 | order ドメインで同様 |
| 6 | `refactor/cost-rename` / `refactor/order-rename` | 4 | **消費者がいる場合のみ**。後方互換ビューを置き、API 契約と突き合わせてエンティティ単位でリネーム → 旧名撤去 |
| 7 | `chore/cleanup-legacy` | 5 | 雛形・残骸 dir・rewrite スクリプト・旧 workflow を削除、`STRUCTURE.md`/`README.md` 更新 |

> 流れの要約: 「足すブランチ（骨格・ネイティブ基盤）→ エンティティ単位で畳むブランチ → （必要なら）リネームブランチ → 掃除ブランチ」。将来のエンティティ追加（invoice 等）は `feature/<entity>-outputs` 1 ブランチで完結し、`outputs/<entity>/` を足すだけになる。

---

## 6. リネームの 2 分岐（消費者の有無）

- **消費者なし**（ドラフト段階で未公開）: Phase 3 の移動と**同時にリネーム**してよい（#4 / #5 に統合）。最も簡素。
- **消費者あり**（API `repository.py` / BI が現テーブルを参照）: 破壊的なので段階適用。
  - 新名（例 `cost_master`）を作成しつつ、旧名のビュー（`config { name: "Dim_COST_MASTER" }` で旧名 view を別途定義し新名を指す）を一時的に残す。
  - 下流を新名へ移行 → 旧名ビュー撤去。これを **1 エンティティずつ**。
  - 破壊的変更は契約変更として扱い、API 側スキーマと突き合わせてから適用する（命名 §6・戦略 §7）。

---

## 7. ロールバック・安全策

- **squash** により「1 PR = 1 revert」。問題があれば該当 PR を revert。
- **build-once**: 直前の安定 tag / commit を release configuration の commitish に固定し直し、再 run で即復旧。
- **fix-forward**: BigQuery テーブルや環境設定を手で直さない。直すのは `main`、dev で再現確認してから昇格。
- 不具合は **再発防止の assertion / テスト**を必ず追加し、dev のゲートで二度と素通りさせない。

---

## 8. 実行チェックリスト

**Phase 0（確認）**
- [ ] `*` / `*-stg` / `*-prd` の `workflow_settings.yaml` を diff（環境差は project だけか／SQL ドリフトはないか）
- [ ] 下流消費者（API/BI）の有無
- [ ] `V_COEP` の中身（declaration か変換ありか）
- [ ] 組織ポリシー `dataform.restrictGitRemotes` 許可リスト

**strict act-as（時限・並行）**
- [ ] 環境ごとのカスタム SA 作成
- [ ] 既定サービスエージェントに各カスタム SA への Service Account User ＋ Token Creator（actAs）
- [ ] クロスプロジェクトのアタッチ権限
- [ ] 既存 repo の段階適用期限（〜2026-07-31）までに全 release/workflow をカスタム SA 実行へ

**完了基準**
- [ ] 単一 Dataform リポジトリ＝単一 Git リポジトリ
- [ ] `definitions/` が sources/intermediate/outputs の 3 層
- [ ] 命名規約（小文字 snake_case・`dim_/fct_` なし）
- [ ] `dev`/`stg`/`prod` release configuration（project override）が稼働、workflow configuration で定期 run
- [ ] `empty/`・`github-actions/`・`develop/`・`staging/`・`dataform-rewrite.sh`・個別 workflow を削除

---

## 参考

- 到達点の運用設計: `dataform-operating-model.md`
- 命名・層構造: `dataform-naming-convention.md`
- Dataform 公式（workflow lifecycle / configure compilations / strict act-as）は運用設計ドキュメントの「参考（一次情報）」を参照
