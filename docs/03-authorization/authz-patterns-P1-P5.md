# 権限制御パターン P1〜P5：メリット / デメリット / 実装・設定 / 手順

> 対象：共通データモデル（例 `mart.dept_cost`）への参照アクセス制御を、BQ Sharing・RLS・CLS・FastAPI・PyCasbin・OPA、Cloud Run / ARO、Entra ID、イントラのユーザーリポジトリ（OpenGIM 等）でどう実装するか。
> 各パターンの実装・設定は 2026 時点の公式ドキュメント等で確認（出典は末尾）。
> 最終更新：2026-06-14

---

## 0. 全パターン共通の前提（最初に押さえる）

- **`mart.dept_cost` は普通のBQテーブル（データ）**。Dataform 等で生成し、権限は **RLS / CLS / Authorized View を後付け**で被せる。「権限管理用マート」という専用オブジェクトは無い。
- **BQ がクエリ時に判定で使える「誰」は `SESSION_USER()`（メール）と IAM グループ所属のみ**。オンプレ照会やリクエスト属性の受け渡しは不可。
- **Entra 利用者を BQ の一級プリンシパルにするのが Workforce Identity Federation（WIF）**。WIF は OAuth 2.0 トークン交換（RFC 8693）に従い、外部IdPの資格情報を STS に渡すと短期の Google Cloud アクセストークンが返る。これで「本人としてクエリ → RLS/CLS が効く」が成立する。
- **属性（本部コード等）は BQ 側に実体化が必要**（mapping 表 or グループ）。federation でオンプレ直参照は不可（→ 別ドキュメント「マッピング表案」）。
- **RLS=行 / CLS=列 / マスキング=値**は独立に合成できる。行ポリシーは「どのレコード」、列タグは「どのフィールド」、マスキングは「フィールドのどこまで」を担い、各層を独立に調整できる。

---

## P1：収束アーキ（BQ を唯一の頭脳・本人クエリ）

**概要**：Entra→WIF→（Cloud Run 上の薄い FastAPI、または BI/SQL）が**本人として**BQをクエリ。RLS/CLS が両チャネルで効く。API は行/列の PDP を持たず、REST↔SQL ＋ ID 仲介だけ。

### メリット
- 強制機構が **BQ の RLS/CLS 一つ**。二重オーサリング・ドリフトが原理的に起きない。
- チャネルA（BQネイティブ）/B（API）が**同じ資源・同じポリシー、違いはプロトコルだけ**。
- 監査・権限が BQ に一元化。RLS/CLS/マスキングを独立に合成できる。
- WIF/Workload Identity で**鍵レス**。

### デメリット
- **表現力の天井**：BQ-SQL で書けない複雑条件や、BQ 以外のデータ源は同じ頭脳で守れない。
- **API 呼び出し毎に BQ クエリ**（レイテンシ/課金）。サブクエリを含む行アクセスポリシーは BigQuery Storage Read API と非互換（単純述語のみ）で、フルアクセスが必要な SA ジョブ等は TRUE フィルタを使う。RLS はパーティション/クラスタのプルーニングに参加しない。
- **本人クエリ用の ID 基盤**（WIF トークン交換・OBO）の構築運用が要る。
- RLS の構造制約：JSON 列に適用不可・ワイルドカードテーブル不可・一時テーブル不可・RLS のあるテーブルを参照するテーブルには適用不可。

### 実装・設定すべきところ
- **WIF**：ワークフォースプール＋プロバイダ（Entra、OIDC/SAML）。`principal://iam.googleapis.com/locations/global/workforcePools/POOL/subject/SUBJECT` 形式で、サインイン設定に audience とトークンエンドポイント（`https://sts.googleapis.com/v1/oauthtoken`）を記述。bq での WIF サポートは gcloud 390.0.0 以降。
- **BQ IAM**：WIF プリンシパル（`principalSet://…`）に `bigquery.user` / `jobUser`。RLS の grantee-list も WIF 識別子で記述。
- **RLS**：`CREATE ROW ACCESS POLICY`（および `IF NOT EXISTS` / `OR REPLACE`）で作成。最初に管理者/SA 向けの全行（TRUE フィルタ）ポリシー、次に業務フィルタのポリシーを置く。`CREATE ROW ACCESS POLICY name ON table GRANT TO ('group:…') FILTER USING (predicate)`。SESSION_USER と mapping 表を使えば DDL でなくデータで権限を管理できる（例：`dept_code IN (SELECT dept_code FROM ref.v_visible_dept WHERE email = SESSION_USER())`）。
- **CLS**：taxonomy を作成（`gcloud data-catalog taxonomies create`）→ ポリシータグ作成 → FINE_GRAINED_ACCESS_CONTROL を有効化 → 列にタグ → `taxonomies policy-tags set-iam-policy` で Fine-Grained Reader を付与。タグ付き列は既定で誰も（管理者でも）読めない。列へのタグ付与はスキーマ更新で行い、CREATE TABLE DDL では指定できない。動的データマスキングも併用可。予算列=タグなし／実績列=タグ＋管理職のみ Reader。
- **ホスト**：Cloud Run に薄い FastAPI（Entra OIDC 検証 → STS トークン交換 → 本人 creds で BQ クライアント）。VPC で BQ 私設接続、サービス自身は Workload Identity。
- **ref 表の同期**：オンプレ→GCS→定期ロード（別ドキュメント参照）。

### 何をするか（手順）
1. WIF プール/プロバイダを Entra で構成し、サインイン設定を配布。
2. ワンマート（Dataform）と `ref` 表（属性・オンプレ同期）を用意。
3. RLS（TRUE＋業務フィルタ、SESSION_USER＋ref 参照）と CLS（taxonomy/tag/Reader）を定義。
4. Cloud Run に FastAPI（OIDC 検証→STS 交換→本人クエリ）をデプロイし VPC 接続。
5. チャネルA（BI/SQL）は WIF identity で同じ mart に直接接続。
6. approved fixtures で A/B 両チャネルの見え方一致を検証。

---

## P2：API-PDP（OPA・部分評価で WHERE を pushdown）

**概要**：FastAPI＋OPA。OPA が**部分評価で WHERE を生成**し、API の BQ クエリに付加。直/sharing 利用者には BQ RLS/CLS で担保（多重防御）。

### メリット
- 脳は 1 つ（Rego）。同じ Compile API リクエストから、生 SQL の WHERE 句や UCAST など複数表現を生成でき、Accept ヘッダで出力形式を選ぶ。
- BQ-SQL で書きづらい複雑条件も Rego で表現でき、移植性（UCAST）も得られる。
- ポリシーは Git/PR で監査（policy-as-code）。

### デメリット
- **WHERE 付加は API チャネル内のみ有効**。直/sharing 購読者は別途 BQ RLS が必須＝結局二箇所（同一 Rego から生成すれば緩和）。
- Enterprise OPA の SQL 生成方言は postgres / mysql / sqlserver / sqliteで、**BigQuery 方言は対象外** → BQ 向けに適応するか UCAST から翻訳が要る。
- OPA は外部データを取りに行けない → 属性は input に積む（PIP）か事前同期。
- OPA サイドカー/バンドル配信など運用要素が増える。

### 実装・設定すべきところ
- **Rego**：`package filters` の include 規則で、未知の値を `input.<TABLE>.<COLUMN>` として指定し、残った条件がフィルタになる。呼び出しは `POST /v1/compile/filters/include` に input と unknowns を渡し、Accept で SQL/UCAST を選択。EOPA は maskRule で列マスクも生成（空フィルタ＝全件、空マスク＝値表示）。
- **変換**：生成 WHERE を BQ 用に適応（方言差・テーブル名）。公式 SDK は targets と tableMappings で列/表名を差し替えて SQL/UCAST を出せる。または `opa-compile-response-parser` 等で AST を自前変換。
- **FastAPI**：リクエスト時に Entra/OpenGIM 属性を input に組み立て（context）→ compile → WHERE 適応 → BQ クエリ（SA 実行可）。
- **BQ**：直/sharing 向けに同等の RLS/CLS（同一 Rego から生成 or 並走）。
- 任意：Permit の PDP は filterObjects で部分評価＋SQL 翻訳を内蔵（自前変換を肩代わり）。

### 何をするか（手順）
1. データフィルタ用 Rego（include / mask）を書く。
2. OPA（または EOPA / Permit PDP）をサイドカー/バンドルで配備。
3. FastAPI に PEP＋compile 呼び出し→WHERE を BQ 方言に適応→BQ クエリを実装。
4. 直/sharing 用に BQ RLS/CLS を（同一ポリシー源から）用意。
5. approved fixtures で API と BQ 両強制の一致を検証。

---

## P3：API-PDP（PyCasbin・粗い認可＋自前 SQL）

**概要**：FastAPI＋PyCasbin。Casbin は**エンドポイント/アクション（粗い）認可**。行/列は自前で WHERE/列選択を構築。直/sharing は BQ RLS/CLS。

### メリット
- 軽量・FastAPI プロセス内に埋め込める。Casbin はアクセス制御モデルを PERM メタモデル（Policy/Effect/Request/Matchers）の CONF に抽象化し、RBAC ロールと ABAC 属性を 1 つのモデルに同居できる。Async は PyCasbin 1.23.0 以降で対応。
- 追加インフラが最小（ライブラリ）。fastapi-authz の CasbinMiddleware は AuthenticationMiddleware と組み合わせて使う。

### デメリット
- **Casbin は SQL WHERE を吐けない**（真偽/許可列挙のみ）→ 行/列は自前実装＝もう一つの真実源になりやすい。
- ABAC の `in` 演算子は Go 版のみ対応で jCasbin・Node-Casbin は未対応等、複雑条件に弱い。
- 直/sharing は Casbin の外＝BQ RLS/CLS 必須。脳が分散する。

### 実装・設定すべきところ
- **model.conf**：PERM（request/policy/effect/matcher）で RBAC＋ABAC 属性を組合せ。
- **ポリシー保存**：SQLAlchemy / async-sqlalchemy / databases / Django ORM などの公式アダプタ。変更伝播は watcher / 再読込。
- **FastAPI**：fastapi-authz（CasbinMiddleware）または fastapi-casbin-auth でエンドポイント/アクションを判定。fastapi-casbin-auth は ACL/RBAC/ABAC をサポート。
- **行/列**：認可結果から自前で WHERE/列マスクを構築し BQ クエリ。
- **BQ**：直/sharing＆多重防御に RLS/CLS。

### 何をするか（手順）
1. model.conf＋ポリシー（DB アダプタ）を定義。
2. FastAPI に CasbinMiddleware を組込み（認証 Middleware と連携）。
3. 行/列の WHERE/列選択ロジックを自前実装。
4. BQ RLS/CLS を用意（直/sharing）。
5. テストで粗い認可＋BQ 強制の整合を確認。

---

## P4：二重実装＋適合テスト

**概要**：API 側（Casbin or OPA）と BQ RLS/CLS を**別々に実装**し、ゴールデン期待値（approved fixtures）を CI で両者に当ててドリフト検出。仕様 1・実行 2。

### メリット
- 既存資産（OPA/Casbin）と BQ ネイティブの両方を活かせる。
- runtime 統一が不要で導入が速い。

### デメリット
- 実装が二重＝乖離リスクが常在（テストで担保する前提）。
- 仕様の単一管理（PAP）が前提条件になる。

### 実装・設定すべきところ
- **正本 spec**（人が読むポリシー）を Git/カタログで一元管理。
- **フィクスチャ**：〈被験者＋文脈 → 見えるべき行/列〉の期待値集合。
- **CI**：API（HTTP）と BQ（SQL）双方に同じフィクスチャを実行し、差分が出たら落とす（ハーネスの計算的センサー）。

### 何をするか（手順）
1. 正本 spec とフィクスチャを作る。
2. API 側・BQ 側をそれぞれ実装。
3. CI に適合テストを組み、差分ゼロをマージゲートにする。

---

## P5：BQ Sharing のみ（API なし）

**概要**：API を作らず、BQ Sharing で linked dataset を配り、RLS/CLS で強制。チャネルA のみ。

### メリット
- 最短・最小。FastAPI/Casbin/OPA 不要。
- Publisher 側 IAM を増やさず購読者がセルフ購読（linked dataset は `bigquery.user` のみで参照）、複製マートも不要。

### デメリット
- プログラム/エージェント（REST）提供は不可。
- BQ ネイティブ（SQL/BI）利用者に限定。

### 実装・設定すべきところ
- **BQ Sharing**：Exchange＋Listing を公開し、購読で読取専用 linked dataset を生成。
- **ワンマート＋RLS/CLS**（＋Authorized View 意味層）。
- **購読者識別**：Entra 利用者は WIF で BQ プリンシパル化し、RLS は**グループ粒度**で効かせる。

### 何をするか（手順）
1. ワンマート＋RLS/CLS を用意。
2. Exchange/Listing を公開し、対象に linked dataset を購読させる。
3. WIF＋グループで RLS を効かせ、見え方を検証。

---

## まとめ：どのパターンがどこに向くか

| パターン | 一言 | 向くケース | 頭脳 |
|---|---|---|---|
| **P1 収束アーキ** | BQ 単一機構・本人クエリ | GCP 寄せで「同一機構・プロトコル差」を最優先。参照系・BQ 完結 | BQ |
| **P2 OPA pushdown** | 単一 Rego で WHERE も生成 | 複雑条件/移植性が要るが単一脳を保ちたい | OPA(API)＋BQ |
| **P3 PyCasbin** | 軽量・粗い認可＋自前SQL | 既に Casbin 資産・粗い認可中心・小規模 | 分散 |
| **P4 二重実装＋適合テスト** | 仕様1・実行2をテストで縛る | 既存二系統を当面活かしつつ安全運用 | spec1/runtime2 |
| **P5 Sharing のみ** | API なし最小構成 | 検証用/最小、BQ ネイティブ消費者のみ | BQ |

**共通の最終防壁**：どのパターンでも、直 SQL / sharing 購読者は API を素通りするため **BQ の RLS/CLS が最後の砦**。API 側 PDP（OPA/Casbin）は「API 経由のみ」有効である点を常に前提にする。

---

## 出典（2026 時点で確認）

1. Google Cloud — Introduction to BigQuery row-level security（grantee/filter、構造制約、Storage Read API 非互換、Admin/DataOwner）. https://docs.cloud.google.com/bigquery/docs/row-level-security-intro
2. Google Cloud — Use row-level security（CREATE ROW ACCESS POLICY DDL、二段ポリシー、List、WIF 識別子）. https://docs.cloud.google.com/bigquery/docs/managing-row-level-security
3. Google Cloud — Using row-level security with other features（TRUE フィルタ、コピー非同期、CLS 併用）. https://docs.cloud.google.com/bigquery/docs/using-row-level-security-with-features
4. Google Cloud — Restrict access with column-level access control（ポリシータグ、スキーマ設定、CREATE TABLE DDL 不可、マスキング）. https://docs.cloud.google.com/bigquery/docs/column-level-security
5. OneUptime — Implement RLS with CLS in BigQuery（taxonomy/policy tag 手順、FINE_GRAINED_ACCESS_CONTROL）. https://oneuptime.com/blog/post/2026-02-17-how-to-implement-row-level-security-policies-in-bigquery-with-column-level-access-controls/view
6. OneUptime — Set up RLS using row access policies（GRANT TO / FILTER USING 例、SESSION_USER＋mapping）. https://oneuptime.com/blog/post/2026-02-17-how-to-set-up-row-level-security-in-bigquery-using-row-access-policies/view
7. OneUptime — Column-level security with policy tags（Fine-Grained Reader 付与、既定で非公開）. https://oneuptime.com/blog/post/2026-02-17-how-to-implement-column-level-security-in-bigquery-with-policy-tags/view
8. adriennevermorel — BigQuery Row Access Policies（RLS/CLS/マスキングの独立合成、TRUE フィルタ）. https://adriennevermorel.com/notes/bigquery-row-access-policies/
9. Google Cloud — REST Resource: rowAccessPolicies（filterPredicate を含む）. https://docs.cloud.google.com/bigquery/docs/reference/rest/v2/rowAccessPolicies
10. Google Cloud — Workforce Identity Federation（RFC 8693 トークン交換）. https://docs.cloud.google.com/iam/docs/workforce-identity-federation
11. Google Cloud — Obtain short-lived tokens for WIF（principal://、STS エンドポイント、gcloud）. https://docs.cloud.google.com/iam/docs/workforce-obtaining-short-lived-credentials
12. Google Cloud — Configure WIF with Microsoft Entra ID（Entra 連携手順）. https://docs.cloud.google.com/iam/docs/workforce-sign-in-microsoft-entra-id
13. Google Cloud — Configure WIF with AWS/Azure（bq は gcloud 390.0.0+ で WIF 対応）. https://docs.cloud.google.com/iam/docs/workload-identity-federation-with-other-clouds
14. Medium（Google Cloud Community）— BigQuery Agents via End User Credentials（外部トークンで利用者権限に準拠）. https://medium.com/google-cloud/enable-bigquery-agents-in-gemini-enterprise-via-end-user-credentials-079b1dc24406
15. Open Policy Agent — REST API（Compile API、SQL/UCAST、Accept ヘッダ）. https://www.openpolicyagent.org/docs/rest-api
16. Open Policy Agent — Data filtering / Partial evaluation（unknowns＝input.TABLE.COLUMN→WHERE）. https://www.openpolicyagent.org/docs/filtering/partial-evaluation
17. Open Policy Agent — Writing valid Data Filtering Policies. https://www.openpolicyagent.org/docs/filtering/fragment
18. Styra — Data Filters Compilation API（SQL 方言 postgres/mysql/sqlserver/sqlite、maskRule）. https://docs.styra.com/enterprise-opa/reference/api-reference/partial-evaluation-api
19. Enterprise OPA CHANGELOG（SQL WHERE / UCAST / SQLite 生成）. https://github.com/open-policy-agent/eopa/blob/main/CHANGELOG.md
20. npm — @open-policy-agent/opa（getMultipleFilters / tableMappings）. https://www.npmjs.com/package/@open-policy-agent/opa
21. Permit.io — Data Filtering（filterObjects・SQL 翻訳内蔵）. https://docs.permit.io/how-to/enforce-permissions/data-filtering/
22. PyCasbin（PERM モデル、RBAC＋ABAC 同居、Async 1.23.0+、`in` 演算子の制約）. https://github.com/pycasbin/casbin / https://pypi.org/project/pycasbin/
23. PyPI — fastapi-authz（CasbinMiddleware）. https://pypi.org/project/fastapi-authz/
24. PyPI — fastapi-casbin-auth（ACL/RBAC/ABAC）. https://pypi.org/project/fastapi-casbin-auth/
25. Microsoft Learn — What's new with Azure Red Hat OpenShift（マネージドID GA 等）. https://learn.microsoft.com/en-us/azure/openshift/azure-redhat-openshift-release-notes
