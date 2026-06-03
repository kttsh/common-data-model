# リポジトリ戦略・運用設計

データプラットフォーム（Dataform / FastAPI / 認可ポリシー）のリポジトリ分割方針、ディレクトリ構成、3 ランドスケープへの昇格・修正フロー、運用の初期設計をまとめたドキュメント。

- 作成日: 2026-05-31
- ステータス: ドラフト（案 B で確定、PDP 埋め込み・3 ランドスケープ前提）
- 言語・FW 前提: **Python / FastAPI 採用決定済み**（`../02-architecture/runtime-framework-decision.md`）

---

## 1. 背景と管理対象

本プラットフォームで管理する成果物と、そのデプロイ先は以下の通り。

| 成果物 | 内容 | デプロイ先 | デプロイ手段 |
|---|---|---|---|
| Dataform | `SQLX`、`workflow_settings.yaml` 等 | BigQuery (GCP) | GitHub Actions |
| FastAPI | BigQuery のテーブルを参照する Python アプリ | Azure App Service | GitHub Actions |
| 認可 | PEP / PDP（ポリシー） | API プロセスに埋め込み | （API と同梱） |
| ドキュメント | スキーマ・モデル検討の Markdown | リポジトリ管理 | — |

API で公開するテーブルは Order / Requisition / Invoice / Man Hour … と継続的に増えていく。この「増やし方」を設計の中心論点とする。

### 設計の軸（最重要）

判断の軸が 2 つあり、これが噛み合わないのが本設計の肝。

- **変更の軸 = ビジネスエンティティ**（Order, Requisition, …）。「Invoice を追加」は Dataform・FastAPI・認可ポリシー・ドキュメントを横断する。
- **デプロイの軸 = 技術 / クラウド**。Dataform→BigQuery(GCP)、FastAPI→Azure App Service と、ランタイムも native 連携も別。

原則: **デプロイ境界 = リポジトリ境界、ドメイン境界 = リポジトリ内のディレクトリ境界**。テーブルが増えてもリポジトリは増やさず、各リポジトリ内のエンティティ単位ディレクトリを足していく。

---

## 2. 調査で効いた制約（要点）

分割判断を左右する事実。

### Dataform
- 推奨ディレクトリは `definitions/` を `sources / intermediate / outputs` に分け、特に **`outputs` はビジネスエンティティ単位でサブディレクトリ**を切る（orders、sales 等）。今回の Order/Invoice… の増やし方と一致。
- core 3.0 以降、設定は `workflow_settings.yaml`（ルート配置）に保存。
- 分割戦略はチーム単位／ドメイン単位／中央 + ドメイン別の 3 通り。クロスリポジトリ依存は data source 宣言で表現し、双方向依存は避ける。
- **native 連携は「単一 Dataform リポジトリ = 単一リモート Git リポジトリ」が前提**。コンソールの workspace / release を使うなら Dataform は専用 Git リポジトリを持つのが自然。
- 分割の主目的はコンパイル資源上限（おおむね 1000 アクション超で危険）回避・権限の細粒度化・可読性。**この閾値に達するまでは 1 リポジトリ内で増やす方が良い**。
- デプロイは GCP-native（release config をブランチに対応付け）でも、配布の CLI/Docker イメージを GitHub Actions で実行する CI/CD でも可能。

### FastAPI
- ファイル種別（routers/models）で分ける構成は小規模向け。多ドメインのモノリスでは **ドメイン単位構成**（各パッケージが router/schemas/models/service/dependencies/exceptions を持つ。Netflix Dispatch 由来）が推奨。
- グローバルな `config.py / database.py / main.py / pagination.py` は `src/` 直下。
- **1 エンティティ = `src/<entity>/` パッケージ 1 式**。

### Azure App Service
- デプロイは「コード方式（Python ランタイム + 起動コマンド `uvicorn main:app`）」と「コンテナ方式（Docker → ACR 経由）」の 2 系統。
- GitHub Actions では Deployment Center 構成で build / deploy の 2 段ワークフロー。認証は service principal シークレットより **OIDC（federated credentials）** が推奨。

### GitHub Actions（モノレポ／複数デプロイ運用）
- 定石は paths フィルタ（変更分のみ起動）、matrix（並列）、reusable workflow（重複排除）、environment + secrets（環境分離）。
- 20 サービス未満なら paths フィルタ + サービス別の薄いワークフロー、それ以上で動的な変更検知に寄せるのが実務的な目安。

---

## 3. リポジトリ分割案

### 案 A: フルモノレポ
1 リポジトリに `dataform/ api/ policies/ contracts/` を同居させ、paths フィルタ + reusable workflow で CI/CD を振り分ける。

- 長所: ドメイン横断変更（Invoice 追加）が 1 PR で原子的に閉じる。スキーマ契約が 1 か所に集約され drift を発見しやすい。
- 短所: Dataform の native 連携は 1 Git = 1 Dataform repo 前提のため、Dataform を CLI/API 方式デプロイに寄せる必要がある（コンソール workspace の利点を一部諦める）。言語・ツールチェーン混在。権限が粗くなる。

### 案 B: プラットフォーム別分割 ★採用
デプロイ先・ランタイムごとにリポジトリを分ける。

- 長所: Dataform の 1 Git = 1 repo 制約に素直。CI/CD・権限・リリース周期が独立。Azure と GCP の native フローを各々最大限活用。データエンジニアとバックエンドの責務分離が自然。
- 短所: Invoice 追加が複数リポジトリ・複数 PR に分散。**スキーマ drift 対策が必須**（§7）。
- 各リポジトリ内ではエンティティ単位ディレクトリで増やすので、リポジトリ数は増えない。

### 案 C: ビジネスドメイン別分割
ドメイン（調達、会計 …）ごとに Dataform スライスと API スライスを同居させる。Dataform のコンパイル上限接近、ドメインごとに権限・スケジュールが大きく分かれる、チーム完全分割、といった段階で意味が出る。今の規模では過剰設計。**B で始めて C へ育てる**のが現実的。

### 判断の目安
- チーム横断の原子的変更・契約一元管理を最優先 → A
- デプロイ独立性・権限分離・各クラウド native 活用を優先 → **B（採用）**
- ドメインが強く分化 / Dataform が約 1000 アクションに接近 → C

### 3.5 世の中の実践による裏付け（なぜ「1 API リポ・ドメイン別ディレクトリ」か）

「請求 / 発注 / 工数 / 原価 … を API 側で分けるか」という問いは、混同しがちな **3 つの直交軸**に分解すると答えが定まる。

| 軸 | 選択肢 | 本設計の決定 |
|---|---|---|
| リポジトリ軸 | monorepo / polyrepo | **デプロイ先で分割**（`dataform` と `api` は別リポ＝案 B） |
| デプロイ軸 | 単一デプロイ / 複数デプロイ | **api は単一デプロイ**（1 App Service） |
| 内部構造軸 | レイヤ別 / **ドメイン別** | **ドメイン別**（`src/<entity>/`） |

エンティティ（Order / Invoice / Man Hour / Cost …）は**内部構造軸**の話であり、**リポジトリを分ける理由にはならない**。同一 `api` リポ内のドメイン別ディレクトリで増やす（＝**モジュラーモノリス**）。一次情報による裏付け:

- **monorepo ≠ monolith、そして今は分割もしない**: FastAPI Best Practices（Netflix Dispatch 由来）は「多ドメインのモノリスは**ファイル種別別でなくドメイン別**ディレクトリに」と明言。各ドメインが router/schemas/service/repository を自己完結で持つ（§4 の構成はこれに準拠）。
- **モジュラーモノリス（Shopify）**: マイクロサービス分割を「分散システムの複雑さ」で却下し、**デプロイ単位を増やさず業務概念でモジュール分割**。ただし境界を CI で強制しないとただのモノリスへ退化する、という教訓付き（→ ドメイン間 import を明示・限定し、`import-linter` 等で境界違反を CI 検出するのが軽量な対策）。
- **分割は便益が正当化されてから（Fowler）**: "MonolithFirst" / "MicroservicePremium"。境界を最初から正しく引くのは難しく、サービス間リファクタはモノリス内より遥かに困難。グリーンフィールドの全面分割は失敗パターン。
- **コンウェイの法則**: サービス分割の便益は「1 サービス＝専任チーム」の自律性から来る。**単一/少数チームが全ドメインを見るなら、デプロイを分けても便益ゼロ・運用コストだけ増える**。

### 3.6 分割トリガー（個別ドメインを別サービス/別リポへ切り出す条件）

将来、特定ドメインだけを `api` から切り出すのは **MicroservicePremium を払う価値が出たとき**のみ。全面分割はしない（案 C「B で始めて C へ育てる」と整合）。

```
そのドメインに専任チームができ、独立リリースサイクルが要る？     ─No→ 分けない（ディレクトリのまま）
        │Yes
そのドメインだけ突出したスケール/負荷/障害分離/コンプラ境界がある？  ─No→ 分けない
        │Yes
デプロイ競合が恒常的ボトルネックになっている？                   ─Yes→ そのドメインだけ "edge" として切り出す
```

切り出しを将来楽にするため、今からドメイン間依存を明示・限定し CI で境界を守る（Shopify "Wedge" の軽量版）。

---

## 4. リポジトリ構成（清書）

PDP を**埋め込み**にしたため、ポリシー本体は API の成果物に同梱される。よってポリシーは api リポジトリ内に置き、`src/authz/policies/` のみ CODEOWNERS でセキュリティ担当所有とする。

### repo: `dataform`（→ BigQuery）

```
dataform/
├── definitions/
│   ├── sources/                    # declare()。共通・最初に1回
│   │   └── <source_system>/…
│   ├── intermediate/               # stg_* 変換ロジック（API非公開）
│   └── outputs/                    # ★API公開面 = エンティティ単位で増やす
│       ├── order/      (order.sqlx + columns doc + assertions)
│       ├── requisition/
│       ├── invoice/
│       └── man_hour/
├── includes/                       # 共通（増やさない）
│   ├── constants.js                #   project/dataset を vars 経由で参照
│   ├── naming.js                   #   命名規約マクロ
│   └── assertions.js               #   共通アサーション
├── workflow_settings.yaml          # defaultProject/Dataset/Location, dataformCoreVersion, vars
├── package.json
├── environments/                   # ★環境差はここだけ（コードは共通）
│   ├── dev.json   (project=pj-dev)
│   ├── stg.json   (project=pj-stg)
│   └── prod.json  (project=pj-prod)
└── .github/workflows/
    ├── ci.yml                      # PR: dataform compile + dry-run + assertions
    ├── deploy-dev.yml              # main push: dev PJ へ run
    └── promote.yml                 # 承認ゲートで stg→prod（同一commit）
```

責務: BigQuery のデータモデル定義・変換・品質保証（assertions）と、その BigQuery へのデプロイ。`outputs/<entity>` が API の公開面に対応する。

### repo: `api`（FastAPI + 埋め込み PEP/PDP → Azure App Service）

```
api/
├── src/
│   ├── main.py / config.py / database.py / pagination.py / dependencies.py   # 共通
│   ├── authz/                      # 認可（共通）
│   │   ├── pep.py                  #   enforcement（FastAPI dependency/middleware）
│   │   ├── engine.py               #   埋め込みPDP（OPA-WASM / Cedar / casbin 等）
│   │   ├── context.py              #   判断属性の組み立て
│   │   └── policies/               #   ★ポリシー本体＝エンティティ単位で増やす
│   │       ├── common.rego         #     テナント分離など共通
│   │       ├── order.rego / invoice.rego / …
│   ├── order/                      # ★エンティティ単位 = 公開テーブル1つに対応
│   │   ├── router.py / schemas.py / service.py
│   │   ├── repository.py           #   outputs/order を読むBQクエリ
│   │   ├── dependencies.py / exceptions.py
│   ├── requisition/ / invoice/ / man_hour/
├── tests/{order/…, authz/}         # authzのポリシーテストを必須化
├── Dockerfile                      # build-once promote のためコンテナ推奨
├── pyproject.toml
└── .github/workflows/
    ├── ci.yml                      # PR: ruff+pytest+policyテスト+image build(検証)
    ├── deploy-dev.yml              # main push: ACR push → dev App Service へ同一digest
    └── promote.yml                 # 承認ゲートで stg→prod（同一digest）
```

責務: HTTP エンドポイント、リクエスト/レスポンス契約（Pydantic）、BigQuery 参照（`repository.py` に集約）、認可の enforcement と評価（埋め込み PDP）。`src/<entity>/` が公開テーブル 1 つ 1 つに対応する。

### repo: `data-contracts`（任意）

責務: スキーマ/モデル検討の Markdown と、機械可読なテーブル契約（列名・型・PII タグ等）。Dataform と API の間の seam（接合面）。採用するなら outputs の列定義と api の schemas の整合を CI で突き合わせる（§7）。

---

## 5. スケーリング: 共通 vs 増やす

テーブルが増えても、**増えるのは各レイヤーの「エンティティ単位ディレクトリ」だけ**。土台は共通化して使い回す。

### 5.0 1 エンティティの三面対応（URL ↔ コード ↔ データ）

1 つのビジネスエンティティは、公開 URL・API コード・Dataform 出力の 3 面で**同じドメイン境界を 1:1 で表現**する。新エンティティ追加はこの 3 面（＋認可ポリシー）に同名で 1 ディレクトリずつ足すだけ。

| エンティティ | URL（公開契約） | API ディレクトリ | Dataform 出力 | 認可ポリシー |
|---|---|---|---|---|
| Invoice | `/v1/invoices` | `src/invoice/` | `definitions/outputs/invoice/` | `src/authz/policies/invoice.rego` |
| Purchase Order | `/v1/purchase-orders` | `src/purchase_order/` | `definitions/outputs/purchase_order/` | `…/purchase_order.rego` |
| Man Hour | `/v1/man-hours` | `src/man_hour/` | `definitions/outputs/man_hour/` | `…/man_hour.rego` |
| Cost | `/v1/costs` | `src/cost/` | `definitions/outputs/cost/` | `…/cost.rego` |

- **命名の使い分け**: URL＝小文字 kebab-case 複数形（`/purchase-orders`）、コード/Dataform ディレクトリ＝snake_case（`purchase_order`）。`man-hour`/`cost` のような不可算寄りの語も原則は複数形 URL（`/man-hours`, `/costs`）とし、単数を選ぶ場合は AIP-122 の例外として全体で統一する。
- `src/<entity>/router.py` は `APIRouter` を定義し、`main.py` の `include_router(..., prefix="/<collection>")` で束ねる。これで 1 アプリ・1 デプロイのまま URL 名前空間とコードがドメインで一致する。


### 増やす（エンティティ追加ごとに加算）
- Dataform: `definitions/outputs/<entity>/`（必要に応じて `sources/`・`intermediate/` のエンティティ固有分）
- FastAPI: `src/<entity>/` パッケージ 1 式（router/schemas/service/repository/…）
- 認可: `src/authz/policies/<entity>.rego`
- 契約/Docs: `<entity>` のスキーマ定義

### 共通（write-once、増やさない・使い回す）
- Dataform: `includes/`（マクロ・命名規約・共通アサーション）、`sources/` 宣言、`workflow_settings.yaml`
- FastAPI: `database.py`（BigQuery クライアント）、`config.py`、基底 Pydantic モデル、`pagination.py`、認証・PEP の共通 dependency、`authz/engine.py`、レスポンス整形
- CI/CD: reusable workflow を 1 本用意し、**エンティティ追加でワークフローファイルを増やさない**（paths フィルタ or matrix で吸収）
- 認可: PDP ランタイム（埋め込みエンジン）、`common.rego`（テナント分離など）

新エンティティ追加時に作るのは `outputs/<entity>/`・`src/<entity>/`・`policies/<entity>.rego` の 3 点セットのみ。これによりレビュー範囲とデプロイ範囲が追加 1 ディレクトリに局在する。

---

## 6. ランドスケープと昇格・修正フロー

### 6.1 ランドスケープ対応（dev / stg / prod）

| 層 | dev | stg | prod |
|---|---|---|---|
| GCP | pj-dev | pj-stg | pj-prod |
| Azure | App Service(dev) | App Service(stg) | App Service(prod) |

- **App Service は landscape ごとに独立インスタンス**（プランも分離。dev は安い SKU 可）。3 ランドスケープと、prod 内の「staging slot による無停止 swap（blue/green）」は別概念。後者は必要なら prod 内にだけ足す。
- 各 App Service が参照する GCP プロジェクトは **GitHub Environments の変数/シークレットで注入**。dev App Service → pj-dev を固定対応。
- 認証は鍵レスで統一: GitHub Actions→GCP は Workload Identity Federation(OIDC)、GitHub Actions→Azure は federated credentials(OIDC)。App Service 実行時の BigQuery 参照認証（Azure→GCP）は実装で詰める論点（WIF か、最小権限 SA の安全な保持か）。

### 6.2 昇格モデルの核となる 2 原則

1. **Build once, promote the same artifact.** dev で検証したのと同一成果物（api はコンテナの同一 digest、Dataform は同一 commit のコンパイル結果）を stg・prod へ動かす。環境差は config 注入だけ。
2. **Fix forward at the source. 環境を直接いじらない。** stg の BigQuery テーブルを手で直す／stg の App Service に入って patch する、は禁止。直すのは常にリポジトリのコード。

### 6.3 昇格フロー（trunk-based + GitHub Environments 承認ゲート）

```
feature → PR(CI) → main
   main push  →  dev 自動デプロイ → 動作確認
   承認        →  stg へ昇格（同一成果物）→ 確認
   承認        →  prod へ昇格（同一成果物）→ tag付与
```

### 6.4 「stg で不備に気付いた場合、どこで何を治すか」

不具合クラスで分ける。

**A. コードの不具合（SQL ロジック / API ロジック / ポリシー）**
- 治す場所: リポジトリ（main から fix ブランチ）。stg は触らない。
- 手順: fix → CI → main → **dev で再現 & 修正確認** → stg へ再昇格 → prod。
- 再発防止のため必ず再現テストを追加（Dataform assertion / pytest / policy テスト）。dev のゲートで二度と素通りさせない。
- Dataform はテーブルなので、不整合テーブルは fix 後に再 run で再生成（incremental は必要なら full refresh）。

**B. 環境設定/インフラの不具合（env var 誤り / IAM 不足 / dataset 名 / App Service 設定）**
- 治す場所: config-as-code（`environments/<env>.json`、GitHub Environment 変数、bicep/terraform）。手動変更しない。
- 手順: 設定変更 → dev で同型確認 → stg 再適用。

**C. データ起因（stg のデータが dev に無かったエッジケースを露出）**
- 本質はコード未対応なので A に帰着。
- 加えて、そのケースを dev でも再現できるテストデータ/assertion を用意し、dev の表現力を stg に追いつかせる。

**例外: 緊急 prod hotfix**

トランクベースでは main が prod より先行して未検証変更を含むことがあるため、hotfix の切り出し元を場合分けする。

- 通常（main HEAD ≒ prod 相当）: `hotfix/*` を main から切る。
- main が先行して未検証変更を含む場合: **prod に出ているコミット（= prod タグ）から `hotfix/*` を切る**。最小修正で新成果物を作り prod へ（dev/stg は fast-track 検証）。完了後に main へ back-merge して dev/stg を退行させない。
- いずれも必ずパイプライン経由。環境（BigQuery テーブルや App Service 実体）の直接編集はしない。

詳細は §10.5 を参照。

### 6.5 なぜこの原則か

Dataform 公式も複数プロジェクトへ展開する際は、リポジトリのコードベースを全プロジェクトで同一に保ち、各コピーのコンパイル・実行のカスタマイズは workspace compilation overrides・release configuration・workflow configuration で行うことを推奨している。環境ごとにコードを分岐させると再現性が壊れるため避ける、という整理。本設計の「全環境でコード同一・差分はオーバーライドで吸収」はこれと同じ思想。

---

## 7. スキーマ契約（分割時の最重要リスク対策）

案 B では Dataform が作る BigQuery テーブルと FastAPI が読む `repository.py` の前提がずれると API が壊れる（schema drift）。seam として以下を行う。

- Dataform 側はテーブルに `columns{}` ドキュメントと assertions を必ず付ける。
- その列定義を機械可読な契約（YAML/JSON）として `data-contracts` に置き、両リポジトリの CI で参照・検証する。
- 契約変更は契約リポジトリの PR を起点にし、Dataform と API 双方の CI を走らせる（破壊的変更の検知）。
- 後方互換を壊す変更（列削除・型変更）は契約変更として段階適用する。

これにより、分割していてもドメイン横断変更の安全性をモノレポ並みに保てる。

---

## 8. 運用の初期メモ（叩き台）

- **ゲート**: GitHub Environments で stg/prod に required reviewers。prod は承認 + tag で変更管理。
- **同時実行**: Actions の `concurrency` で同一環境への多重デプロイを抑止。
- **可観測性**: stg は本番相当の監視を入れ「本番で初めて気付く」を減らす。Dataform=assertions、API=health check + ログ/メトリクス。
- **ロールバック**: build-once なので「前の digest / 前の tag を再デプロイ」で即復旧。
- **Dataform のデプロイ方式**: promotion 重視なら CLI 方式（Actions が CLI で対象 PJ の BigQuery に run）が「同一 commit→dev→stg→prod」を最も素直に表現できる。GCP ネイティブのスケジューリング/監視 UI が欲しい場合は API 起動の管理リポジトリ方式も可。スケジューリング（定期リフレッシュ）はデプロイとは別決定として後で詰める。
- **CODEOWNERS**: `api/src/authz/policies/` をセキュリティ担当所有に。

---

## 9. オープン事項 / 次のアクション

- [ ] `deploy-dev.yml` / `promote.yml` の雛形作成（OIDC・承認ゲート・同一成果物昇格込み）を dataform / api それぞれで。
- [ ] CLI 方式での Dataform 環境オーバーライドの具体フラグを、導入する `@dataform/cli` のバージョンに合わせて確定。
- [ ] App Service 実行時の Azure→GCP 認証方式（WIF か最小権限 SA か）の決定。
- [ ] 埋め込み PDP の評価エンジン選定（OPA-WASM / Cedar / casbin 等）と `context.py` の入力属性設計。
- [ ] `data-contracts` を採用するか、契約検証を api/dataform の CI に内製するかの決定。
- [ ] Dataform のスケジューリング（定期リフレッシュ）方式の決定（Actions cron / Cloud Scheduler / native workflow config）。

---

## 10. ブランチ戦略

§6 の昇格モデル（trunk-based + GitHub Environments 承認ゲート、build-once / fix-forward）と整合させるため、**全リポジトリをトランクベースで統一**する。`main` を唯一のトランク（保護ブランチ）とし、そこから短命の feature ブランチを切って PR で戻す。

### 10.1 共通基盤（全リポジトリ共通）

- **ブランチ命名**: `feature/<entity>-<desc>`、`fix/<desc>`、`chore/<desc>`、`hotfix/<desc>`
- **main 保護**: PR 必須、CI green 必須、レビュー 1 名以上（該当箇所は CODEOWNERS）、直 push 禁止、force push / 削除禁止、stale approval は自動 dismiss
- **マージ方式は squash**: main を線形に保ち「1 PR = 1 revert 単位」にする。ロールバックとタグ運用が素直になる
- **環境展開はブランチではなく昇格**: main マージ → dev 自動、承認ゲート → stg → prod。同一成果物（api は同一 digest、Dataform は同一 commit）を動かす
- **タグ**: prod 到達コミットに付与（build-once 成果物の人間可読エイリアス。技術的な昇格キーは git SHA）

非採用判断: **branch-per-environment（`dev`/`stg`/`prod` ブランチ）は採らない**。環境ごとのコード分岐を誘発し、build-once / fix-forward を壊すため。Dataform の native 連携はこれを強制しがちだが、CLI 方式デプロイにすることで回避する。

### 10.2 repo: `dataform`

共通基盤そのまま。開発を Dataform の workspace で行う場合、**workspace は短命の feature ブランチを追跡**し、commit/push 後に PR で main へ（workspace ≒ feature ブランチ）。ローカル / CLI 開発でも同型。

- feature 例: `feature/invoice-outputs`
- PR CI: `dataform compile` + dry-run + assertions
- main マージ → dev プロジェクトで `run`。昇格は同一 commit を stg → prod
- feature 開発時の BigQuery 書き込みは dev プロジェクト内でスキーマ接尾辞等により分離し、他開発者・prod テーブルと衝突させない
- タグ: `dataform-vX.Y.Z`

### 10.3 repo: `api`

共通基盤そのまま。コンテナの build-once-promote 前提。

- feature 例: `feature/invoice-endpoint`
- PR CI: ruff + pytest + **policy テスト** + image build（検証のみ、push しない）
- main マージ → image を ACR に immutable tag（git SHA）で push し、その digest を dev App Service へ。昇格は同一 digest を stg → prod
- `src/authz/policies/` 変更は CODEOWNERS（セキュリティ担当）レビュー必須
- タグ: `api-vX.Y.Z`

### 10.4 repo: `data-contracts`（採用する場合）

トランクベースは同じだが、Dataform と api の双方に消費されるため**バージョニングが主役**になる。

- PR CI: 契約スキーマの検証 + できれば消費側（dataform / api）の互換チェック
- semver で管理し、breaking / non-breaking を明示。breaking はメジャー up のタグで示し、消費側の追従は別 PR で段階的に
- レビューはデータ責任者を CODEOWNERS に

### 10.5 hotfix の扱い（精密版）

トランクベースでは main が prod より先行して未検証変更を含むことがあるため、hotfix の切り出し元を場合分けする。

- **通常（main HEAD ≒ prod 相当）**: `hotfix/*` を main から切る
- **main が先行して未検証変更を含む場合**: **prod に出ているコミット（= prod タグ）から `hotfix/*` を切る**。最小修正で新成果物を作り prod へ（dev/stg は fast-track 検証）。完了後に main へ back-merge して dev/stg を退行させない
- いずれも必ずパイプライン経由。環境（BigQuery テーブルや App Service 実体）の直接編集はしない

### 10.6 リポジトリ間の協調

案 B で各リポジトリは独立したバージョン線を持つ。「Invoice 追加」のように複数リポジトリへまたがる変更は、共有ブランチではなく **contract のバージョン + §9 のオープン事項プロセス**で協調する（contract bump → dataform 追従 PR → api 追従 PR）。

---

## 参考（一次情報）

- Dataform: Best practices for repositories — https://docs.cloud.google.com/dataform/docs/best-practices-repositories
- Dataform: Best practices for the workflow lifecycle / code lifecycle — https://docs.cloud.google.com/dataform/docs/managing-code-lifecycle
- Dataform: Manage a repository（workflow_settings.yaml）— https://docs.cloud.google.com/dataform/docs/manage-repository
- FastAPI Best Practices（ドメイン単位構成）— https://github.com/zhanymkanov/fastapi-best-practices
- Azure App Service: Deploy Python FastAPI（GitHub Actions）— https://learn.microsoft.com/en-us/azure/app-service/tutorial-python-postgresql-app-fastapi
- GitHub Actions: monorepo の paths フィルタ / reusable workflows のパターン（各種実務ガイド）
- Martin Fowler "MonolithFirst" — https://martinfowler.com/bliki/MonolithFirst.html
- Martin Fowler "Conway's Law" — https://martinfowler.com/bliki/ConwaysLaw.html
- Shopify Engineering: Deconstructing the Monolith（モジュラーモノリス）— https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity
- monorepo ≠ monolith の整理 — https://monorepo.tools/
