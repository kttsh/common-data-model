# PyCasbin Basic Specification

> ステータス: 調査メモ（PyCasbin 採用後の基本仕様整理）
> 情報時点: 2026年6月
> Sources: PyCasbin README, Casbin official docs `overview`, `syntax-for-models`, `supported-models`, `abac`, `rbac-with-domains`, `function`, `adapters`, `data-permissions`, `rbac-api`
> Related: `policy-examples-purchase-order.md`, `row-scope-to-bigquery-implementation.md`, `single-record-final-check.md`, `../authorization-strategy.md`, `../authorization-boundaries-and-interface.md`, `../row-level-filtering-layering.md`, `../shared-pdp-across-api-and-bigquery.md`, `../../04-research/abac-authz-library-comparison.md`

---

## 1. 要約

PyCasbin は Casbin の Python 実装で、FastAPI プロセス内に組み込める認可ライブラリである。モデルファイル（CONF）で認可ロジックを定義し、ポリシー（CSV / DB / adapter）を読み込んだ `Enforcer` が `enforce()` で許可/拒否を判定する。

本プロジェクトでは、PyCasbin を **粗い gate（モデル/エンドポイント単位の allow/deny）と RBAC/ABAC 判定**の中核として使う。行レベル WHERE 生成や列マスキングは PyCasbin のネイティブ責務ではないため、既存方針どおり `Decision` の `row_filter` / `masked_columns` を返す自前層で補完する。

---

## 2. PyCasbin が担うもの / 担わないもの

PyCasbin が担うもの:

- `{subject, object, action}` または任意に拡張したリクエスト形式の認可判定。
- ACL、RBAC、RBAC with domains、ABAC、RESTful path/method、deny-override、priority などのモデル表現。
- ロール階層、ユーザー・ロール対応、ロール・ロール対応の管理。
- model / policy の読み込みと、adapter 経由の policy 永続化。
- Management API / RBAC API による policy の参照・追加・削除。
- `keyMatch`, `keyMatch2`, `regexMatch`, `ipMatch`, `globMatch` などの matcher 関数。

PyCasbin が担わないもの:

- 認証。Entra ID / OIDC / JWT 検証は別レイヤで行う。
- ユーザー台帳・ロール台帳そのものの管理。Open-GIM 等の正本は別に持つ。
- BigQuery 向け SQL WHERE 句の生成。
- 列レベルマスキングの適用。
- Open-GIM など外部 PIP からの属性取得。

---

## 3. 基本アーキテクチャ

PyCasbin の実行単位は `Enforcer` である。`Enforcer` は model と policy を読み込み、アプリケーションから渡されたリクエストを matcher で評価する。

最小構成:

```python
import casbin

e = casbin.Enforcer("path/to/model.conf", "path/to/policy.csv")

if e.enforce(subject, object, action):
    ...
else:
    ...
```

FastAPI では、ハンドラ内に直接散らすのではなく、`authz_core` のような共通認可モジュールに閉じ込めるのが望ましい。

```python
decision = authz.authorize(principal, "read", Resource(model=model))
if decision.effect == "deny":
    raise HTTPException(status_code=403, detail=decision.reason)

where, params = compile_to_bigquery(decision.row_filter)
```

PyCasbin は上記のうち、主に `authorize()` 内の allow/deny 判定を担う。`row_filter` と `masked_columns` は、PyCasbin の判定結果・Open-GIM 属性・モデルメタデータを使って本プロジェクト側で組み立てる。

---

## 4. PERM モデル構文

Casbin の model CONF は PERM メタモデルで構成される。

- `request_definition`: `enforce()` に渡す入力の形を定義する。標準は `r = sub, obj, act`。
- `policy_definition`: policy 行の構造を定義する。標準は `p = sub, obj, act`。
- `policy_effect`: 複数 policy が一致したときの合成方法を定義する。
- `matchers`: request と policy をどう照合するかを式で定義する。
- `role_definition`: RBAC を使う場合に追加する。標準は `g = _, _`、domain 付きは `g = _, _, _`。

> 注意（PyCasbin の対応セクション）: Casbin 公式の Model Syntax には separation of duties 等を表す任意セクション `[constraint_definition]` があるが、**PyCasbin 本体は未実装**である。`casbin/model/model.py` の `Model.section_name_map` が認識するのは `r`（request）/ `p`（policy）/ `g`（role）/ `e`（effect）/ `m`（matchers）の **5 セクションのみ**で、`load_model` も constraint をロードしない（情報時点: 2026年6月）。SoD のような「ロール付与時の制約」を担保したい場合は、PyCasbin の model に書くのではなく、ロール付与フロー側のバリデーションなどアプリケーション層で実装する。

ACL の最小例:

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

対応する policy:

```csv
p, alice, data1, read
p, bob, data2, write
```

`policy_effect` は任意式ではなく、Casbin が用意する組み込み effect から選ぶ。代表例は allow-override、deny-override、allow-and-deny、priority、subject-priority である。

注意点:

- policy の各要素は文字列として扱われる。
- matcher の評価順は性能に影響する。安い条件（例: `r.obj == p.obj`）を先に置き、ロール探索など高コストな条件を後ろに回す。
- PyCasbin の matcher 評価器は `simpleeval` 系である。言語実装間の完全互換を前提にしすぎず、公式に説明される構文に寄せる。

---

## 5. 対応モデル

本プロジェクトで主に関係するモデル:

- ACL: subject / object / action の単純一致。
- RBAC: user-role / role-role の階層で権限を束ねる。
- RBAC with domains: tenant / workspace / company などの domain ごとに role を分ける。
- ABAC: subject / object / action の属性を matcher で参照する。
- RESTful: path pattern と HTTP method を認可対象にする。
- Deny-override: allow と deny が同時に一致した場合に deny を優先する。
- Priority: firewall ルールのように policy の優先順位で判定する。

今回の FastAPI + BigQuery API では、入口の API 判定は RESTful + RBAC/ABAC、モデルデータの閲覧条件は ABAC + 自前 filter 生成に寄せるのが自然である。

---

## 6. RBAC / RBAC with Domains

通常の RBAC は `g = _, _` で user-role / role-role の関係を表す。

```ini
[role_definition]
g = _, _

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

domain 付き RBAC は `g = _, _, _` とし、第三引数に domain を渡す。

```ini
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _, _, _

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.act
```

policy 例:

```csv
p, admin, tenant1, data1, read
p, admin, tenant2, data2, read
g, alice, admin, tenant1
g, alice, user, tenant2
```

本プロジェクトで domain を使う場合、候補は `company`, `business_unit`, `department`, `cost_center`, `model_owner` などである。ただし、domain は role のスコープを切るための軸であり、行フィルタの全条件を domain に押し込まない。複雑な所属・役職・データ属性は ABAC または自前 `row_filter` 生成で扱う。

---

## 7. ABAC

ABAC では、`enforce()` に渡す subject / object を文字列ではなく属性アクセス可能な Python オブジェクトとして扱い、その属性を matcher から参照する。

概念例:

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = r.sub.department == r.obj.department && r.obj.name == p.obj && r.act == p.act
```

ABAC の重要な制約:

- 属性参照できるのは request 側（`r.sub`, `r.obj`, `r.act`）である。
- policy 側（`p.sub` など）に構造体やクラスを直接入れることはできない。
- 複雑な条件を policy に逃がす場合は `eval()` を使う。

`eval()` を使う例:

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub_rule, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = eval(p.sub_rule) && r.obj.name == p.obj && r.act == p.act
```

policy 例:

```csv
p, r.sub.job_level >= 5 && r.sub.department == r.obj.department, department_cost, read
p, r.sub.role == "finance_admin", department_cost, read
```

本プロジェクトでは `r.sub` に Open-GIM から得た所属・役職・原価センタ等を入れ、`r.obj` にモデル名・モデル所有部門・機密区分等を入れる。PyCasbin は「この request を許可するか」を判定し、どの BigQuery 行へ絞るかは別の filter 生成層で扱う。

---

## 8. RESTful path / HTTP method

RESTful API の認可では、object に path、action に HTTP method または業務 action を入れる。

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && keyMatch2(r.obj, p.obj) && r.act == p.act
```

policy 例:

```csv
p, model_reader, /models/:model/records, GET
g, alice, model_reader
```

FastAPI 側では、HTTP method をそのまま使うか、`GET -> read`, `POST -> create`, `PUT/PATCH -> update`, `DELETE -> delete` のような業務 action に正規化するかを先に決める。既存メモの方針に合わせるなら、API 経路と抽出経路で語彙を共有しやすい `read` などの業務 action に寄せるのがよい。

---

## 9. Policy storage / adapters

PyCasbin は policy を adapter 経由で読み書きする。開発初期は CSV ファイルで十分だが、運用時に UI や申請フローから policy を更新するなら DB adapter を検討する。

Python 向けに確認できた代表 adapter:

- built-in File Adapter
- Django ORM Adapter
- SQLObject Adapter
- SQLAlchemy Adapter
- Async SQLAlchemy Adapter
- Async Databases Adapter
- Peewee Adapter
- SQLModel Adapter
- MongoEngine / Pony ORM / Tortoise ORM などの third-party adapter

PyCasbin README では `AsyncEnforcer` も案内されている。FastAPI が async I/O 中心で、policy を非同期 DB adapter から読む構成にする場合は `AsyncEnforcer` を検討する。

```python
e = casbin.AsyncEnforcer("path/to/model.conf", adapter)
await e.load_policy()
```

運用上の判断:

- policy を Git 管理するなら File Adapter + デプロイ反映が単純。
- 管理画面・申請ワークフローで policy を更新するなら SQLAlchemy / Async SQLAlchemy adapter が候補。
- 複数インスタンスで policy 更新を即時反映するなら Watcher / Dispatcher 相当の同期機構も別途検討する。

---

## 10. Management API / RBAC API / data permissions

PyCasbin は policy を実行時に参照・変更する API を持つ。

代表的な RBAC API:

```python
roles = e.get_roles_for_user("alice")
users = e.get_users_for_role("data1_admin")
has = e.has_role_for_user("alice", "data1_admin")
e.add_role_for_user("alice", "data2_admin")
e.delete_role_for_user("alice", "data1_admin")
permissions = e.get_permissions_for_user("alice")
implicit_roles = e.get_implicit_roles_for_user("alice")
implicit_permissions = e.get_implicit_permissions_for_user("alice")
```

データ権限に関係する機能:

- `get_implicit_permissions_for_user()` は、直接付与だけでなくロール継承で得た権限も含めて取得する。
- Batch API は複数リクエストをまとめて判定し、データセットのフィルタリング補助に使える。

ただし、これは「候補データに対して多数の allow/deny を判定する」ための機能であり、BigQuery に押し下げる WHERE 句を自動生成するものではない。大量データでは全件取得後に batch 判定するのではなく、PyCasbin の判定結果や policy から本プロジェクト側の条件 AST を作り、BigQuery の WHERE へ変換する。

---

## 11. 本プロジェクトでの使い方

推奨する責務分担:

- PyCasbin: endpoint / model 単位の allow/deny、RBAC、ABAC 条件評価。
- Open-GIM 連携層: subject 属性の取得、短 TTL キャッシュ、属性の正規化。
- `authz_core`: PyCasbin を隠蔽し、`authorize(subject, action, resource) -> Decision` を提供。
- filter 生成層: `Decision.row_filter` を作る。BigQuery 固有ではなく条件 AST として保持する。
- BigQuery 翻訳層: 条件 AST をパラメータ化 WHERE へ変換する。
- FastAPI PEP: deny なら 403、allow なら WHERE 適用、列マスク、監査ログ記録。

最初の PoC では、PyCasbin の policy を「粗い gate」と「行フィルタ種別の選択」に使うのが現実的である。

例（`p = role, obj, act, filter_key` のような独自 `policy_definition` を切る想定）:

```csv
p, finance_manager, department_cost, read, own_department
p, finance_admin, department_cost, read, all_departments
g, alice, finance_manager
g, bob, finance_admin
```

この policy を直接 SQL に変換するのではなく、`own_department` / `all_departments` のような統制された filter key を本プロジェクト側で解釈し、`row_filter` を生成する。これにより、モデルオーナーに自由 SQL を書かせず、レビュー可能な条件語彙に閉じ込められる。

---

## 12. ポリシー定義例（別ドキュメント）

注文伝票 `purchase_order` を題材にした、PyCasbin 初心者向けの policy 記述例は [`policy-examples-purchase-order.md`](policy-examples-purchase-order.md) に切り出した。主経路（一覧フィルタ）を「基本的な仕組み → 実例 → 補足 → BigQuery への展開のイメージ」の順で示している。込み入った話は 2 つの別紙に分けた。

- `row_scope` を BigQuery の WHERE に落とす込み入った Python 実装 → 別紙 [`row-scope-to-bigquery-implementation.md`](row-scope-to-bigquery-implementation.md)。
- 詳細画面で 1 件開いたときの最終チェック（1 件単位の `enforce()`） → 別紙 [`single-record-final-check.md`](single-record-final-check.md)。

扱うケースは次のとおり。

- 日本居住者以外は何も取得できない
- 自分が起票した伝票だけ見る
- 自部門で他人が起票した伝票だけ見る
- グループ長以上だけ決裁金額を見せる
- 請負社員の拒否
- 関係外秘の伝票は本人と承認者だけが見られる
- （更新系は本メモのスコープ外。更新 API の設計が固まってから別途整理する）

---

## 13. 設計上の注意

PyCasbin を使う場合も、既存の認可方針は維持する。

- 認可属性の正本は Entra ID ではなく Open-GIM。
- PyCasbin は FastAPI 内の PDP 実装候補であり、APIM の代替ではない。
- 行・列制御は PyCasbin だけで完結しない。`Decision` で filter / mask を返し、FastAPI が BigQuery とレスポンス整形に適用する。
- policy の表現力を広げすぎない。`eval()` に複雑な式を大量に入れると、レビュー・テスト・性能の負担が増える。
- matcher はテスト必須。特に deny-override、priority、domain、ABAC 属性欠落のケースを単体テストで固定する。
- policy 更新を実行時に許可する場合は、監査ログ、承認フロー、ロールバック、複数インスタンス同期を合わせて設計する。

---

## 14. 次に決めること

1. `subject` に載せる Open-GIM 属性の最小セット。
2. `resource` の標準形。例: `model`, `path`, `owner_department`, `sensitivity`, `tenant`。
3. action 語彙。HTTP method か、`read/create/update/delete/export` などの業務 action か。
4. policy 保存方式。初期は Git 管理 CSV、将来は SQLAlchemy adapter へ移すか。
5. filter key / condition AST の仕様。自由 SQL ではなく、統制された語彙で表現する。
6. PoC 対象モデル。既存メモどおり「部門費モデル」を第一候補にする。

