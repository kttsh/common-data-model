# Dataform 既存リポジトリ改造プラン

`ai-driven-comod-cdem`（全ブランチを統合した `main`）を、`../02-architecture/repository-strategy.md` で定めた**あるべき姿**へ移行する手順。破壊を避ける**順番**と、トランクベースの**ブランチステップ**をまとめる。

- 作成日: 2026-06-03
- 改訂日: 2026-06-04（`../02-architecture/repository-strategy.md` 準拠へ全面改稿。ネイティブ前提を CLI 路線へ変更）
- ステータス: ドラフト（実行手順）
- 関連: `../02-architecture/repository-strategy.md`（上位設計：リポジトリ戦略・3 ランドスケープ昇格・運用・ブランチ戦略）、`dataform-naming-convention.md`（命名・層）
- 前提: **CLI 路線**（ローカル CLI 手打ち → Step 2 で GitHub Actions の CLI 実行）。GCP 側 Dataform リポジトリは作らず BigQuery 直叩き。環境差はデプロイ時の **CLI コンパイラオプション**（`--default-database` 等）で注入。認証は**鍵レス**（WIF＋ADC）。
- 旧前提からの変更: かつての「GCP ネイティブ（release configuration のプロジェクトオーバーライド＋strict act-as カスタム SA）」は採用しない。`../02-architecture/repository-strategy.md` §8（運用の初期メモ＝Dataform は CLI 方式）が CLI 路線を採ったため、release/workflow configuration と strict act-as カスタム SA は本プランから撤去した。GCP ネイティブ案の記録は `dataform-operating-model.md`（不採用）に残す。

---

## 1. 現状（as-is）とのギャップ

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
| `-stg` / `-prd` を物理コピーで分離 | 環境差はフォルダ/ブランチで分けず、デプロイ時の CLI オプションで注入 | 戦略 §3・§5 |
| 各テーブルが個別 `workflow_settings.yaml`＝実質別 repo | 1 Git = 1 Dataform repo、`outputs/<entity>/` で増やす | 戦略 §1・§3・§4 |
| `outputs` しか無い（`sources`/`intermediate` 無し） | `definitions/` を 3 層に分割 | 戦略 §3、命名 §2 |
| `dataform-rewrite.sh`（`sed` 置換）＋ REST `writeFile` | `workflow_settings.yaml` は環境非依存、環境差は CLI コンパイラオプションで注入 | 戦略 §3 |
| SA JSON キー＋既定サービスエージェント | 鍵レス：WIF(OIDC)＋ADC。`.df-credentials.json` は非コミット | 戦略 §5 |
| `empty/`・`github-actions/`・`develop/`・`staging/` | 雛形・残骸。`outputs/<entity>/` を1ファイル足す運用 | 戦略 §1・§4 |

> 注: かつて as-is ギャップに挙げていた「ネイティブ repo＋strict act-as カスタム SA」関連は、CLI 路線では GCP 側 Dataform リポジトリを作らないため**論点ごと消滅**する。組織ポリシー `dataform.restrictGitRemotes`（ネイティブ repo の Git リモート制限）も CLI 路線では無関係。

---

## 2. あるべき姿（to-be）— 要約

単一リポジトリ・3 層構造・小文字 snake_case・環境差は CLI コンパイラオプション・鍵レス（WIF＋ADC）。リポジトリ完成形は `../02-architecture/repository-strategy.md` §3 の `dataform` リポジトリ構成を参照。

本プランは**プラットフォーム別分割（`dataform` / `api` / `data-contracts`）のうち `dataform` リポジトリ単体**の移行を対象とする（戦略 §2）。下流（API `repository.py` / BI）との契約突き合わせは、採用するなら `data-contracts` 経由で行う（戦略 §6）。

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
2. **環境コピーの集約は、BigQuery テーブル名を変えなければ下流（API `repository.py` / BI）に無傷。** だから先に畳んで、最も痛い「3 重コピー・ドリフト・cherry-pick 地獄」を消す。環境差は畳んだあと CLI オプション（`--default-database`）で吸収する。
3. **リネームは契約変更として扱う。** 大文字＋`Dim_`/`Fact_` → 小文字 snake_case はテーブル名が変わるので、消費者がいる場合は後方互換ビューを置き、**エンティティ単位**で段階適用する（命名 §6・戦略 §6）。
4. **構造集約（Phase 0〜4）は Step 1 の手動 run で完結させ、自動デプロイ／昇格（Step 2）・Terraform 化（Step 3）はそのあとに分離する**（戦略 ロードマップ）。

---

## 4. フェーズ（順番）と Step マッピング

`../02-architecture/repository-strategy.md` のロードマップ（Step 1 手動 → Step 2 CLI/Actions → Step 3 Terraform）に従い、本プランの Phase 0〜5 は **Step 1（手動・ローカル CLI 手打ち）で構造集約を完結**させる。CI 自動デプロイ・stg/prod 昇格・WIF・Terraform は Step 2／Step 3 に分離する。

### Phase 0 — 確認・準備（破壊なし）【Step 1】
- 目的: 事故要因の洗い出し。
- 作業: ① `*` / `*-stg` / `*-prd` の `workflow_settings.yaml` と `.sqlx` を **diff** し、環境差が「project/dataset 名だけ」か「SQL ロジックも違う（ドリフト）」かを判定（ドリフトしていれば **prod を真**として正す）。② 下流消費者（API/BI）の有無を確認。③ `V_COEP` の中身を確認し `sources` か `intermediate` かを決める。④ dev プロジェクトの**最小 IAM**（compile 時の dry-run／run に必要な BigQuery 権限）と、proxy-only egress 下での `HTTPS_PROXY`・npm registry・`bigquery.googleapis.com` 到達性を確認。
- 検証: インベントリ表が揃い、リネーム方式（§6 の分岐）が決まる。ローカル CLI で `bigquery.googleapis.com` に到達できる。

### Phase 1 — 単一リポジトリ骨格（ローカル CLI で dev 検証）【Step 1】
- 目的: 到達点の器を用意する（既存ディレクトリと共存可）。
- 作業: ルートに**環境非依存の単一 `workflow_settings.yaml`**（`defaultProject`=dev 基準・`defaultLocation`・`dataformCoreVersion` を固定ピン・`vars` は共通のみ）、`definitions/{sources,intermediate,outputs}/`、`includes/`（`constants.js`/`naming.js`/`assertions.js`）、`.github/workflows/ci.yml` を新設。GCP 側 Dataform リポジトリは**作らない**。ローカル CLI で BigQuery を直叩きし、`--default-database=<pj-dev>` を手で渡して dev で空コンパイル／run が通ることを確認する。
- 検証: `dataform compile` ＋ dry-run が dev の最小 IAM で通る（**compile は BigQuery に当たるためオフラインでは完結しない**点に注意）。ローカル CLI run が成功する。
- 備考: 環境差を持ち込んだ `environments/*.json` は**コミットしない**。Step 1 では開発者が `--default-database`（必要に応じ `--schema-suffix` / `--vars`）を手で渡す。

### Phase 2 — 環境コピーを畳む（名前は維持）【Step 1】
- 目的: per-env コピーの解消。
- 作業: `*` / `*-stg` / `*-prd` の 3 コピーを 1 本に集約。環境差は**デプロイ時の CLI コンパイラオプション**（`--default-database=<pj-env>`）で吸収する設計に移し、`dataform-rewrite.sh`（`sed` 置換）と REST `writeFile` を廃止。**テーブル名は現状維持（大文字のまま）**。
- 検証: ローカル CLI で `--default-database=<pj-dev>` を渡して dev に run し、畳む前と同一テーブルが再生成される。同一 commit に対し `--default-database` だけを stg→prod に差し替えて run し、結果が一致することを確認（build-once の素振り）。
- 非破壊性: BigQuery 上の名前を変えないため下流に影響しない。

### Phase 3 — 3 層構造へ再配置（名前は維持）【Step 1】
- 目的: `definitions/` の標準化。
- 作業: `V_COEP` を `sources/sap/`（または `intermediate/stg_coep`）へ。master/detail を `outputs/<domain>/`（`cost/`・`order/`）へ移動。**まだリネームしない**。
- 検証: 各移動をローカル CLI で dev に run → 検証。
- 備考: Phase 2 と 3 はエンティティ単位で 1 ブランチにまとめると効率的（§5 参照）。

### Phase 4 — 命名規約への移行（破壊的・エンティティ単位）【Step 1】
- 目的: 小文字 snake_case 化。
- 作業: §6 の分岐に従う（消費者なしなら Phase 3 と同時、ありなら後方互換ビュー経由で段階移行）。
- 検証: 旧名・新名の両方を参照テストで確認してから旧名を撤去。

### Phase 5 — クリーンアップ【Step 1 末尾】
- 目的: 残骸の除去と仕上げ。
- 作業: `empty/`・`github-actions/`・`develop/`・`staging/`、各ディレクトリの `dataform-rewrite.sh` と個別 `workflows_dataform_write.yml`、ルートの旧 workflow を削除。`STRUCTURE.md`/`README.md` を新構造に更新。旧大文字テーブルが残っていれば契約移行完了後に DROP。
- 検証: 単一リポジトリ＝単一 Dataform repo、3 層、命名規約が満たされ、ローカル CLI run で dev/stg/prod 全てが同一 commit から再生成できる。

### （以降）Step 2 — CLI 自動デプロイ・昇格【本プランの範囲外・参照のみ】
- 構造集約（Phase 0〜5）の完了後に着手。`deploy-dev.yml`（main push → dev へ CLI run）、`promote.yml`（承認ゲートで同一 commit を stg→prod）を追加。**WIF＋環境別 CI 用 SA＋IAM をここで必須化**し、GitHub Environments の承認ゲートと、OIDC subject の環境スコープ制限を入れる（戦略 §5・§7）。

### （以降）Step 3 — Terraform 化【本プランの範囲外・参照のみ】
- デプロイ機構は CLI のまま。dataset・SA・WIF・IAM を `import` でコード化（リスクの高い WIF＋IAM スライスから先行、GCS backend、オンプレ実行）（戦略 §7）。

---

## 5. ブランチステップ（トランクベース）

### 共通ルール
- `main` を唯一の保護ブランチ。PR 必須・CI green 必須・直 push 禁止・**squash マージ**（1 PR = 1 revert 単位）。
- 命名: `feature/<entity>-<desc>` / `chore/<desc>` / `fix/<desc>` / `refactor/<desc>` / `hotfix/<desc>`（戦略 §8）。
- 各 PR の CI は `dataform compile` ＋ dry-run ＋ assertions（**dry-run は BigQuery に当たる**ため、CI ランナーに dev の最小 IAM と `bigquery.googleapis.com` 到達性が必要）。
- マージ後の流れ（Step 1）: `main` にマージ後、開発者が**ローカル CLI** で `--default-database=<pj-dev>` を渡して dev に run → 検証。stg/prod へは同一 commit に対し `--default-database` だけ差し替えて run（自動昇格は Step 2 で `promote.yml` 化）。
- feature 開発時の dev 書き込みは `--schema-suffix` 等で他開発者・prod と分離する（戦略 §8）。

### 順序付きブランチ

| # | ブランチ | フェーズ | 内容 |
|---|---|---|---|
| 1 | `chore/repo-skeleton` | 1 | 環境非依存の単一 `workflow_settings.yaml`・`definitions/{sources,intermediate,outputs}`・`includes/`・`ci.yml` を新設（既存 dir と共存）。ローカル CLI で dev 空 run を確認 |
| 2 | `feature/sap-coep-source` | 3 | `V_COEP` を `sources/sap/coep.sqlx`（または `intermediate/stg_coep`）へ。生 SAP は declaration |
| 3 | `feature/cost-outputs` | 2＋3 | `Dim_COST_MASTER{,-stg,-prd}`＋`Fact_COST_DETAIL{,-stg,-prd}` を `outputs/cost/{cost_master,cost_detail}.sqlx` に集約。名前維持（消費者なしならここでリネーム） |
| 4 | `feature/order-outputs` | 2＋3 | order ドメインで同様 |
| 5 | `refactor/cost-rename` / `refactor/order-rename` | 4 | **消費者がいる場合のみ**。後方互換ビューを置き、API 契約と突き合わせてエンティティ単位でリネーム → 旧名撤去 |
| 6 | `chore/cleanup-legacy` | 5 | 雛形・残骸 dir・rewrite スクリプト・旧 workflow を削除、`STRUCTURE.md`/`README.md` 更新 |

> 流れの要約: 「足すブランチ（骨格）→ エンティティ単位で畳む＋3 層化するブランチ → （必要なら）リネームブランチ → 掃除ブランチ」。旧プランの `chore/native-deploy-setup`（ネイティブ基盤・カスタム SA・release config）は CLI 路線では不要のため削除した。自動デプロイ（`deploy-dev.yml` / `promote.yml`）と WIF は Step 2 で別途追加する。将来のエンティティ追加（invoice 等）は `feature/<entity>-outputs` 1 ブランチで完結し、`outputs/<entity>/` を足すだけになる。

---

## 6. リネームの 2 分岐（消費者の有無）

- **消費者なし**（ドラフト段階で未公開）: Phase 3 の移動と**同時にリネーム**してよい（#3 / #4 に統合）。最も簡素。
- **消費者あり**（API `repository.py` / BI が現テーブルを参照）: 破壊的なので段階適用。
  - 新名（例 `cost_master`）を作成しつつ、旧名のビュー（`config { name: "Dim_COST_MASTER" }` で旧名 view を別途定義し新名を指す）を一時的に残す。
  - 下流を新名へ移行 → 旧名ビュー撤去。これを **1 エンティティずつ**。
  - 破壊的変更は契約変更として扱い、`data-contracts`（採用時）の契約・API 側スキーマと突き合わせてから適用する（命名 §6・戦略 §6）。

---

## 7. ロールバック・安全策

- **squash** により「1 PR = 1 revert」。問題があれば該当 PR を revert。
- **build-once**: Dataform は「同一 commit を同一 `dataformCoreVersion` でコンパイルした結果」を各環境に run することで再現する（戦略 §5）。前の安定 commit / tag を再 run すれば即復旧。
- **fix-forward**: BigQuery テーブルや環境設定を手で直さない。直すのは `main`、dev で再現確認してから昇格（戦略 §5）。
- 不具合は **再発防止の assertion / テスト**を必ず追加し、dev のゲートで二度と素通りさせない。

---

## 8. 実行チェックリスト

**Phase 0（確認）【Step 1】**
- [ ] `*` / `*-stg` / `*-prd` の `workflow_settings.yaml` を diff（環境差は project だけか／SQL ドリフトはないか）
- [ ] 下流消費者（API/BI）の有無
- [ ] `V_COEP` の中身（declaration か変換ありか）
- [ ] dev の最小 IAM（compile の dry-run／run に必要な BigQuery 権限）
- [ ] proxy-only egress 下の `HTTPS_PROXY`・npm registry・`bigquery.googleapis.com` 到達性

**認証（鍵レス）【Step 1 はローカル ADC、必須化は Step 2】**
- [ ] ローカル開発は ADC（`gcloud auth application-default login` 等）で run。`.df-credentials.json` は**コミットしない**
- [ ] Step 2 で GitHub Actions→GCP の **WIF(OIDC)＋環境別 CI 用 SA** を整備（dev/stg/prod を 1 つの万能 SA に集約しない）
- [ ] prod 書き込みは承認ゲート付き `promote.yml` のジョブにだけ束ねる／OIDC subject を環境スコープに絞る

**完了基準【Step 1 構造集約】**
- [ ] 単一 Dataform リポジトリ＝単一 Git リポジトリ
- [ ] `definitions/` が sources/intermediate/outputs の 3 層
- [ ] 命名規約（小文字 snake_case・`dim_/fct_` なし）
- [ ] `workflow_settings.yaml` は環境非依存。環境差は CLI の `--default-database`（必要に応じ `--schema-suffix`/`--vars`）で注入。`environments/*.json` は非コミット
- [ ] 同一 commit から dev/stg/prod を再生成できる（build-once の素振りを確認）
- [ ] `empty/`・`github-actions/`・`develop/`・`staging/`・`dataform-rewrite.sh`・個別 workflow を削除

---

## 参考

- 上位設計: `../02-architecture/repository-strategy.md`（分割案 §3、構成 §4、スケーリング §5、昇格・修正フロー §6、契約 §7、運用 §8、ブランチ §10）
- 命名・層構造: `dataform-naming-convention.md`
- Dataform 一次情報（CLI／コンパイラオプション／strict act-as 等）は `../02-architecture/repository-strategy.md` の「参考」を参照
