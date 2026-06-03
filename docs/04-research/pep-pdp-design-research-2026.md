# PEP / PDP 設計 調査と実装パターン（2026 年版）

> スコープ: 認可機構（PEP/PDP）の世の中のプラクティス、自前実装とライブラリの比較、代表エンジンのポリシー記述例
> 情報時点: 2026 年 5 月
> 前提: 認証=Entra ID、認可属性=OpenGIM、データ=BigQuery、API=FastAPI/App Service、入口=Azure API Management

---

## 目次

1. [調査サマリ（2026 年の世の中のプラクティス）](#1-調査サマリ2026-年の世の中のプラクティス)
2. [「SQL ライクにポリシーを定義」の整理](#2-sql-ライクにポリシーを定義の整理)
3. [代表的なエンジンのポリシー記述例（調達ドメイン）](#3-代表的なエンジンのポリシー記述例調達ドメイン)
4. [参考リンク](#4-参考リンク)

---

## 1. 調査サマリ（2026 年の世の中のプラクティス）

### 1.0 一目で（結論先出し）

| 観点 | 2026 年のコンセンサス |
|---|---|
| 標準プロトコル | **AuthZEN Authorization API 1.0** が 2026 年 3 月 11 日に OpenID Foundation の Standards Track として正式公開（最終版 4 月 29 日）。PEP↔PDP の通信が業界標準化された |
| アーキテクチャ | OWASP 推奨は依然「**Centralized pattern with embedded PDP**」。ただし「組み込みライブラリ／別プロセス／別ホスト」は配備オプションとして個別議論する流れに更新 |
| 自前実装 vs ライブラリ | **PEP は自前、PDP は既製エンジン**が標準解。PDP を自作するのはアンチパターン |
| 行レベル | 「ポリシー → 部分評価 → AST → DB フィルタ」が定石。Cerbos `PlanResources` / OPA `Compile API` が双璧 |
| AI エージェント | 「Authorization Fabric（PEP+PDP）」がエージェント実行ガバナンスの中核に。Gartner も 2025 IAM Summit でエンタープライズに AuthZEN 採用を推奨 |

### 1.1 2026 年に確定したこと

#### AuthZEN 1.0 の正式化

- Authorization API 1.0 が 2026 年 3 月 11 日に Standards Track として正式公開、最終仕様は 2026 年 4 月 29 日付。
- HTTPS バインディングが必須。判定拒否（policy denial）は `200 + decision: false` で、認証失敗（`401`）と明確に区別する設計。
- Identiverse 2024 のデモで「1 つの PEP 実装 → Topaz / Axiomatics / OpenFGA など 5 エンジン以上の PDP をエンドポイント URL の付け替えだけで切替」がライブで実証された。
- Gartner が 2025 IAM Summit Executive Summary でエンタープライズに AuthZEN 採用を正式推奨（ベンダーロックイン回避）。Kong / Envoy / Zuplo などの API Gateway も AuthZEN PEP 機能を実装し始めている。
- OPA も `/access/v1/evaluation` を AuthZEN 1.0 準拠で提供する作業が進行中、Keycloak も 26.7.0 で experimental サポートを追加。

**含意**: 「PEP↔PDP の境界を AuthZEN で書いておく」ことで、埋め込み起点 → 将来別 PDP への乗り換えが現実的な選択肢になった。

#### OWASP Microservices Cheat Sheet の更新（2025〜2026）

OWASP は従来「Centralized with embedded PDP」と「Centralized with external PDP」を別パターンとして並べていたが、両者を 1 つのパターンに統合し、PDP のデプロイ／統合戦略は別セクションで論じる方針に改訂。PDP は (a) ライブラリ埋め込み（例: Casbin）、(b) ローカルサイドカープロセス（例: OPA）、(c) 中央集権の外部 PDP のいずれでも、同じパターンの実装形態として並列に扱える、という整理。

#### AWS Verified Permissions の値下げと Cedar の浸透

AWS Verified Permissions が単発認可 API（IsAuthorized, IsAuthorizedWithToken）の価格を最大 97% 引き下げ、$5 / 100 万リクエストに改定。マネージド PDP が現実的な選択肢になりつつある（ただし Azure 中心の構成には適合しない）。

#### ReBAC（OpenFGA / SpiceDB）が本番普及

Google Zanzibar インスパイアのシステム（SpiceDB / OpenFGA / Permify）が学術的好奇心から本番インフラへ移行。OpenAI は ChatGPT Enterprise のコネクタで SpiceDB を採用し数百億件のきめ細かい権限を捌いている。

#### AI エージェントが新たな主戦場

エージェント認可は「OAuth が答えるのは『エージェントはこの API を呼べるか』であり、『業務ポリシー・コンプライアンス・データ境界・承認しきい値の下でこのアクションを実行すべきか』ではない」。そこで PEP（任意のツール／アクション呼び出し前のゲートキーパー）と PDP（RBAC+ABAC+承認ポリシー評価）の Authorization Fabric が共有エンタープライズ制御プレーンとして必要になる。判定出力は **ALLOW / DENY / REQUIRE_APPROVAL / MASK**。

### 1.2 アーキテクチャパターン（2026 年版）

#### 配備オプションの三段階

| パターン | 形態 | レイテンシ | ポリシー集中管理 | 障害伝播 | 代表例 |
|---|---|---|---|---|---|
| **埋め込みライブラリ (in-process)** | PDP がアプリと同一プロセス | 〜数 μs | 弱（バンドル配布で補う） | プロセス内のみ | Casbin、Cedar SDK、py-abac、Cerbos ePDP |
| **サイドカー** | 同一ホスト別プロセス（localhost） | 1–5 ms | 中 | 同一ホスト | OPA サイドカー（k8s 標準）、SpiceDB 埋め込み |
| **集中型外部 PDP** | 別サービス（HTTP/gRPC） | 5–50 ms（要キャッシュ） | 強 | ネットワーク依存 | Cerbos / OPA 別ホスト、AWS Verified Permissions、Permit.io、Oso Cloud |

OWASP は推奨パターンとして「Centralized with embedded PDP」を挙げており、Netflix の事例（Policy Portal / Repository → Distributor → 各サービスの埋め込み PDP に非同期配信）が代表的な参照アーキテクチャ。

App Service 構成（コンテナ不可）では **サイドカー不可** なので、選択肢は実質「埋め込み」or「別ホスト PDP（短 TTL キャッシュ）」に絞られる。

#### PEP の責務（自前実装する側）

PEP は PDP に比べてシンプルで、認可ロジックを持たない。認可要求を送り、判定結果を評価するだけ。例えば PDP が boolean の allow/deny を返すなら、PEP は HTTP 200/403 にマップする程度。複雑なケースでは PDP が返す JSON を評価し必要なロジックを実装する。

PEP に持たせるべきは:
1. 認証コンテキスト（JWT 検証結果）から AuthZEN の Subject を組み立てる
2. リクエストから Action / Resource / Context を抽出する
3. PDP を呼ぶ（キャッシュ込み）
4. Decision を「短絡 403」「行フィルタ翻訳」「列マスク適用」「監査ログ」に分配する

PEP 自体はアプリ言語で自前実装するのが標準。PDP 側のクライアント SDK を薄くラップする程度。

#### PDP は自作するな

| 自作の落とし穴 | 既製品で得られるもの |
|---|---|
| ポリシー言語の評価器を 1 から実装 | Cerbos: YAML+CEL、OPA: Rego の成熟した評価器 |
| 部分評価（query plan）のアルゴリズム実装 | PlanResources / Compile API として実装済 |
| ポリシーテスト・lint・playground | Cerbos CLI / OPA test / Cedar CLI が完備 |
| バージョニング・配布パイプライン | Bundle API / Git バンドル等が標準 |
| 形式検証・安全性証明 | Cedar の verification-guided development |
| 監査・観測性 | デフォルトで decision log を JSON で吐く |

### 1.3 ライブラリ比較（2026 年版）

#### 主要エンジン一覧

| ライブラリ | ライセンス | モデル | ポリシー言語 | 部分評価/Query Plan | AuthZEN 1.0 | デプロイ | Python 統合 |
|---|---|---|---|---|---|---|---|
| **Cerbos** | Apache 2.0 | RBAC/ABAC | YAML + CEL | ◎ `PlanResources` | ○ | 別プロセス／ePDP | SDK あり、SQLAlchemy アダプタあり |
| **OPA** | Apache 2.0 | 何でも（汎用） | Rego (Datalog 系) | ◎ `Compile API` | ◎（公式対応進行中） | サイドカー／別ホスト | REST 経由、SDK あり |
| **Cedar / AWS Verified Permissions** | Apache 2.0（Cedar） | RBAC/ABAC | Cedar | ○ | ○ | ライブラリ／マネージド（AWS） | SDK あり |
| **OpenFGA** | Apache 2.0（CNCF） | ReBAC（Zanzibar） | OpenFGA DSL | △（関係クエリ） | ○ | 別プロセス | SDK あり |
| **SpiceDB (Authzed)** | Apache 2.0 | ReBAC（Zanzibar） | SpiceDB スキーマ | △ | ○ | 別プロセス／マネージド | SDK あり |
| **Casbin (PyCasbin)** | Apache 2.0 | ACL/RBAC/ABAC/ReBAC | CONF + マッチャ式 | × | × | 埋め込み（プロセス内） | ネイティブ、`fastapi-casbin-auth` 等 |
| **Oso** | Apache 2.0／商用 | RBAC/ABAC/ReBAC | Polar | ○ | △ | ライブラリ／マネージド | SDK あり |
| **py-abac / Vakt** | Apache 2.0 | ABAC（XACML 系） | JSON | △ | × | 埋め込み | ネイティブ Python |
| **Keycloak (AuthZEN PDP)** | Apache 2.0 | RBAC/ABAC | Keycloak Authorization Services | - | ○（実験） | 別サービス | REST |

#### 主要エンジンの詳細評価

**Cerbos**
- 行レベルフィルタ（条件 AST）、列マスキング、列アクセスチェックを 1 つのポリシーで宣言可能。
- アダプタは Prisma / Drizzle / SQLAlchemy / Convex / ChromaDB（RAG）等に拡大。
- 強み: 開発者 UX（YAML + CLI + Playground）、ステートレス、サブミリ秒レイテンシ、ePDP で配布可。
- 弱み: 汎用ではないので Kubernetes Admission 等の用途には不向き。**BigQuery アダプタは公式に無く自作必要**。

**OPA**
- 強み: CNCF 卒業、Kubernetes 標準、Rego の表現力。
- 弱み: Rego は Datalog 系の構文で学習コスト大。本番運用にはポリシー更新配信、バージョン管理、ステータス集約モニタリングを自前で整備する必要がある。
- AuthZEN 対応進行中で、既存 Rego をそのまま使える PDP として将来性は高い。

**Cedar / AWS Verified Permissions**
- 強み: verification-guided development、宣言性の高い構文、Cognito 連携。
- 弱み: AWS 寄り、閉域 Azure 構成には適合しにくい。

**OpenFGA / SpiceDB**
- ReBAC（関係ベース）専用。OpenAI が SpiceDB で数百億件の権限を捌いている実績。
- 「役職・所属・居住地などの属性で行/列を出し分ける」要件は ABAC で素直に表現可能。ReBAC は「起票者単位の権限」「組織ツリー巡回」が支配的になってきた将来の選択肢。

**Casbin (PyCasbin)**
- 強み: 完全埋め込み・低レイテンシ・追加サービス不要・多言語対応・FastAPI 統合。
- 弱み: 行レベルフィルタの **WHERE 生成は弱い**。Cerbos/OPA のような Query Plan 機能がない。

**Oso**
- 強み: Polar 言語の表現力、データフィルタリング機能。
- 弱み: 商用化が進み Oso Cloud（マネージド）中心の戦略。自己ホスト前提だと選択肢としての勢いが弱まっている。

### 1.4 自前実装 vs ライブラリ（決定指針）

#### PEP は必ず自前

| 理由 | 詳細 |
|---|---|
| アプリ固有のコンテキスト抽出が必要 | JWT クレーム → AuthZEN Subject の組み立て、リソース ID 抽出、Context（IP、時刻、デバイス等）構築は各アプリ言語・FW で書くしかない |
| Decision の執行はアプリ層 | 行フィルタ AST → BigQuery SQL 翻訳、列マスク適用、監査ログは PEP の責務 |
| 軽量 | PEP のコード量は通常 200〜500 行程度に収まる |

#### PDP は必ず既製品

PDP の正しさはエンジニアリング難題。評価ロジック、ポリシー言語、部分評価、テストハーネス、監査ログ のフルセットを自作するのはコスト・リスクの両面で割に合わない。

**例外**: 真に小規模（〜数十ルール）かつ将来も拡張しない用途では `py-abac` のような軽量 ABAC ライブラリを「PDP として使う」のはあり。それでも評価器を自作するのは推奨されない。

#### 行レベルフィルタ生成層は半自作

これがプロジェクト最大の自作ポイント。

- **既製品**: Cerbos `PlanResources` または OPA `Compile API` が「ポリシー → 抽象構文木（AST）」を返す
- **自作**: 「AST → BigQuery のパラメータ化 SQL（`@param`、`UNNEST(@array)`）」の翻訳層

設計上の重要ポイント:
- DB-pushable な部分とアプリ層で評価する部分が混在する場合、アダプタは AST を分割し、push 可能な子ノードは filter に、それ以外は postFilter に分配する。
- 認可条件で使う列にインデックス（BigQuery ならパーティション/クラスタ）を必ず張る。

### 1.5 キャッシュ戦略（PDP 呼び出し最適化）

外部 PDP 構成では避けて通れない論点。

実装ガイド:
- **キー**: `(subject_id, action, resource_kind, resource_id?)` + 属性スナップショットのハッシュ
- **TTL**: 60 秒〜5 分（属性鮮度要件で決定）
- **無効化**: 属性ソース（OpenGIM）変更時に明示 invalidate するか、TTL で諦める
- **PlanResources のキャッシュ**: フィルタ AST 自体は subject 属性ごとにキャッシュ可（1 ユーザー = 1 セッションで使い回し）

重要操作（データエクスポート、機密性の高いユーザーアクション）は毎回 PDP に問い合わせ、ルーチン操作は非同期更新付きのキャッシュ済みポリシーを使う、というハイブリッドが一般的。

---

## 2. 「SQL ライクにポリシーを定義」の整理

> 質問: PyCasbin、Cerbos、OPA はいずれも「SQL ライクにポリシーを定義」できるツールか？
> 答え: **合っていない**。混同があるので整理する。

### 2.1 これらのツールの「ポリシー定義言語」は SQL ライクではない

| ツール | ポリシー定義言語 | 雰囲気 |
|---|---|---|
| PyCasbin | CONF ファイル（PERM メタモデル）+ マッチャ式 | 独自の DSL |
| Cerbos | YAML + **CEL**（Common Expression Language） | Google 由来の式言語。`request.principal.attr.dept == request.resource.attr.dept` のような書き方 |
| OPA | **Rego**（Datalog 系） | Prolog 寄り。SQL とは構文・思想が別物 |

どれも SQL ライクではない。Rego に至っては SQL とは思想が真逆（事実とルールで導出するロジックプログラミング）。

### 2.2 ただし役割の本質は別ルートで応えている

役割メモの「SQL ライク（WHERE, SELECT）に条件を設定」は、実は **2 つの異なる要求が混ざっている**:

| 要求 | 中身 | これらのツールの答え |
|---|---|---|
| (a) ポリシーを **SQL の構文で書きたい** | オーナーが WHERE 文を書く感覚で権限ルールを書く | × ── 独自 DSL で書くことになる |
| (b) 結果として **行レベルは WHERE、列レベルは SELECT 相当**で効かせたい | 行絞り込みは BigQuery に WHERE で押し下げ、列はマスク | ◎ ── Cerbos `PlanResources` / OPA `Compile API` が「ポリシー → WHERE 句の AST」を生成する |

つまり「**書くときは独自 DSL、出力されるものは SQL の WHERE 句**」という分業。

### 2.3 既存ドキュメントはどちらに振れているか

既存方針は **意図的に (a) を捨てて (b) に振っている**:

> 自由 SQL ではなく、ポリシーから機械的にフィルタを生成する形にするのが 2026 年の定石

これは役割メモの「SQL ライクに条件設定」をそのまま実装すると **SQL インジェクション・レビュー困難・テスト困難** という問題が出るためで、論点③（自由 SQL の統制）に対応している。

### 2.4 もし本当に「SQL で書きたい」を貫くなら

別ジャンルのツールになる:

- **PostgreSQL Row Security Policies**（`CREATE POLICY ... USING (条件式)`）── SQL ネイティブだが BigQuery には使えない
- **BigQuery 行アクセスポリシー**（`CREATE ROW ACCESS POLICY`）── SQL で書けるが、単一 SA で叩く構成では成立しないため不採用
- **Data API Builder** ── 既に不採用決定

### 2.5 結論

| 質問 | 答え |
|---|---|
| PyCasbin / Cerbos / OPA は SQL ライクにポリシーを定義できる？ | **いいえ**。独自 DSL で書く |
| 役割メモの「SQL ライクに WHERE/SELECT」要件を満たす？ | **半分はい**。ポリシーは独自 DSL で書くが、生成される結果は SQL の WHERE 句 |

議論で握るべきは「**書き味として SQL ライクが欲しいのか、出力として SQL WHERE が出てくれば良いのか**」の切り分け。

---

## 3. 代表的なエンジンのポリシー記述例（調達ドメイン）

### 3.0 前提モデル（共通）

```
Subject（principal）         Resource（purchase_order）
  id: "u_alice"               id: "PO-001"
  department_id: "FIN-A"      creator_id: "u_bob"
  role: "manager"             department_id: "FIN-A"
  subordinate_ids:            amount: 1_500_000
    ["u_bob", "u_carol"]
```

判定したい 3 ルール:
1. **自分が起票** … `resource.creator_id == subject.id`
2. **自分の部門** … `resource.department_id == subject.department_id`
3. **自分の部下が起票** … `resource.creator_id ∈ subject.subordinate_ids`

### 3.1 PyCasbin（埋め込み・ABAC モデル）

**model.conf**

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub_rule, obj, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = eval(p.sub_rule) && r.obj.kind == p.obj && r.act == p.act
```

**policy.csv**

```csv
p, r.obj.creator_id == r.sub.id,                              purchase_order, read
p, r.obj.department_id == r.sub.department_id,                purchase_order, read
p, r.obj.creator_id in (r.sub.subordinate_ids),               purchase_order, read
```

**呼び出し（FastAPI 側）**

```python
e = casbin.Enforcer("model.conf", "policy.csv")
subject = {"id": "u_alice", "department_id": "FIN-A",
           "subordinate_ids": ["u_bob", "u_carol"]}
resource = {"kind": "purchase_order", "creator_id": "u_bob",
            "department_id": "FIN-A"}
e.enforce(subject, resource, "read")   # True
```

**特徴と限界**

- 単一レコード判定（CheckResources 相当）には向く。レイテンシは μs オーダー。
- `in` 演算子を policy.csv で使うには Casbin の関数登録（`AddFunction`）が必要なケースが多い。
- **Query Plan に相当する機能は無い**ので「u_alice が見られる PO 一覧を 1 クエリで取りたい」には別の仕組みが要る。行レベルフィルタ生成用途には弱い。

### 3.2 Cerbos（埋め込み or 別ホスト、YAML + CEL）

**policies/purchase_order.yaml**

```yaml
apiVersion: api.cerbos.dev/v1
resourcePolicy:
  resource: "purchase_order"
  version: "default"
  rules:
    - name: own-orders
      actions: ["read"]
      effect: EFFECT_ALLOW
      roles: ["user"]
      condition:
        match:
          expr: request.resource.attr.creator_id == request.principal.id

    - name: same-department
      actions: ["read"]
      effect: EFFECT_ALLOW
      roles: ["user"]
      condition:
        match:
          expr: >
            request.resource.attr.department_id ==
            request.principal.attr.department_id

    - name: subordinate-orders
      actions: ["read"]
      effect: EFFECT_ALLOW
      roles: ["manager"]
      condition:
        match:
          expr: >
            request.resource.attr.creator_id in
            request.principal.attr.subordinate_ids
```

**呼び出し（CheckResources）**

```python
from cerbos.sdk.client import CerbosClient
from cerbos.sdk.model import Principal, Resource

with CerbosClient("http://cerbos:3592") as c:
    principal = Principal("u_alice", roles={"user", "manager"},
        attr={"department_id": "FIN-A",
              "subordinate_ids": ["u_bob", "u_carol"]})
    resource = Resource("PO-001", "purchase_order",
        attr={"creator_id": "u_bob", "department_id": "FIN-A"})
    decision = c.check_resources(principal, [(resource, ["read"])])
```

**特徴**

- ルールごとに `name` を付けられる → **監査ログで「どのルールで許可されたか」が残る**。
- `manager` ロール限定で部下ルールが効くなど、ロールと属性を混ぜやすい。
- `PlanResources` を呼べば一覧用の AST が返る（後述）。

### 3.3 OPA / Rego（汎用ポリシーエンジン）

**policies/procurement.rego**

```rego
package procurement.purchase_order

import rego.v1

default allow := false

# 自分が起票した PO
allow if {
    input.action == "read"
    input.resource.creator_id == input.subject.id
}

# 自分の部門の PO
allow if {
    input.action == "read"
    input.resource.department_id == input.subject.department_id
}

# 自分の部下が起票した PO（manager のみ）
allow if {
    input.action == "read"
    "manager" in input.subject.roles
    input.resource.creator_id in input.subject.subordinate_ids
}
```

**呼び出し（HTTP POST）**

```python
import httpx
r = httpx.post("http://opa:8181/v1/data/procurement/purchase_order/allow",
    json={"input": {
        "action": "read",
        "subject": {"id": "u_alice", "department_id": "FIN-A",
                    "roles": ["user", "manager"],
                    "subordinate_ids": ["u_bob", "u_carol"]},
        "resource": {"creator_id": "u_bob", "department_id": "FIN-A"}}})
```

**特徴**

- Rego の OR は「同じ名前のルールを複数並べる」で表現する。慣れると簡潔だが、初見では戸惑う書き方。
- 単体テストが充実（`opa test`）。
- 大量ルール時の評価最適化や Compile API による部分評価は強力。

### 3.4 （比較）OpenFGA ── ReBAC で書くとどう違うか

Casbin/Cerbos/OPA はいずれも **属性比較**（「`subject.dept == resource.dept` か」）でルールを書いた。OpenFGA は **関係グラフ**（「subject と resource が関係 X で繋がっているか」）で書く。

**スキーマ**

```
model
  schema 1.1

type user
  relations
    define manager: [user]

type department
  relations
    define member: [user]

type purchase_order
  relations
    define creator: [user]
    define dept: [department]
    define viewer: creator
                 or member from dept
                 or manager from creator
```

**データ（タプル）を入れる**

```
(user:u_bob,            manager,  user:u_alice)   # bob の上司は alice
(department:FIN-A,      member,   user:u_alice)
(department:FIN-A,      member,   user:u_bob)
(purchase_order:PO-001, creator,  user:u_bob)
(purchase_order:PO-001, dept,     department:FIN-A)
```

**判定**

```
check(user:u_alice, viewer, purchase_order:PO-001) → true
  理由: alice は FIN-A の member、もしくは bob (creator) の manager
```

**ここがポイント**

- 「部下が起票した PO」が**最も自然に書ける**のは ReBAC。`manager from creator` の 1 行で済む。
- 一方、「部門所属」も含めて**全部を ReBAC で表現するには `(user, member, department)` のタプルを OpenGIM から OpenFGA に常時同期する必要がある**。属性が動的に変わる組織には運用負荷が高い。
- 既存ドキュメントの結論（**ABAC 中核 + 起票者/組織が支配的になってきたら ReBAC 部分採用**）はまさにここで効く。

### 3.5 Cerbos / OPA は「ポリシー → WHERE 句」へどう化けるか

「u_alice が読める PO を一覧したい」場合、CheckResources を 1 件ずつ呼ぶのではなく **PlanResources**（Cerbos）/ **Compile API**（OPA）を 1 回呼ぶ。

**Cerbos PlanResources のレスポンス（抜粋・概念図）**

```json
{
  "filter": {
    "kind": "KIND_CONDITIONAL_PERMIT",
    "condition": {
      "operator": "or",
      "operands": [
        {"op": "eq", "lhs": "R.attr.creator_id",    "rhs": "u_alice"},
        {"op": "eq", "lhs": "R.attr.department_id", "rhs": "FIN-A"},
        {"op": "in", "lhs": "R.attr.creator_id",
                     "rhs": ["u_bob", "u_carol"]}
      ]
    }
  }
}
```

**FastAPI 側の翻訳層が BigQuery のパラメータ化 SQL に変換**

```sql
SELECT po_id, vendor, amount, ordered_at, ...
FROM   procurement.purchase_orders
WHERE  creator_id     = @subject_id
   OR  department_id  = @subject_dept
   OR  creator_id IN UNNEST(@subordinate_ids)
```

```python
params = {
    "subject_id":      "u_alice",
    "subject_dept":    "FIN-A",
    "subordinate_ids": ["u_bob", "u_carol"],
}
```

これが「**自由 SQL は禁止、ポリシーから機械的に WHERE を生成して BigQuery に押し下げる**」の具体形。オーナーが書くのは YAML（or Rego）だけで、SQL は誰も手で書かない ── ここが安全性とレビュー性の源泉になる。

### 3.6 まとめ表

| ケース | PyCasbin | Cerbos | OPA(Rego) | OpenFGA |
|---|---|---|---|---|
| 自分が起票 | `r.obj.creator_id == r.sub.id` | `creator_id == principal.id` | `creator_id == subject.id` | `creator` 関係を直接持つ |
| 自分の部門 | `r.obj.department_id == r.sub.department_id` | 同形式 | 同形式 | `member from dept` |
| 部下が起票 | `r.obj.creator_id in (r.sub.subordinate_ids)`（要関数登録） | `creator_id in subordinate_ids` | `creator_id in subject.subordinate_ids` | **`manager from creator` 一行で最も自然** |
| 一覧用 WHERE 生成 | × | ◎ PlanResources | ◎ Compile API | △（関係クエリ） |
| 監査ログでルール特定 | △ | ◎（`name` で残る） | ○ | ○ |

書き味の好みで言えば **Cerbos の YAML が最も「ポリシーオーナーに見せやすい」**、最も柔軟なのは **OPA の Rego**、最軽量は **Casbin**、関係性で支配的なら **OpenFGA**。

---

## 4. 参考リンク

### 標準・仕様

- AuthZEN Authorization API 1.0 公式: https://openid.github.io/authzen/
- AuthZEN 1.0 ディープダイブ（2026/04）: https://dev.to/kanywst/authzen-authorization-api-10-deep-dive-the-standard-api-that-separates-authorization-decisions-1m2a
- OPA AuthZEN 対応 Issue: https://github.com/open-policy-agent/opa/issues/8449
- Keycloak AuthZEN 実験サポート: https://www.keycloak.org/2026/05/authzen-as-experimental-feature
- AuthZEN + SSF ファンダメンタル（2026/03）: https://andrewdoering.org/blog/2026/authzen-shared-signals-framework-part-1-fundamentals/
- Auth0 AuthZEN 実装ガイド（FastAPI 例含む）: https://auth0.com/blog/implementing-authzen-guide-openid-authorization-api/
- An Introduction to AuthZEN（Curity）: https://curity.io/resources/learn/authzen/

### エンジン比較・選定

- Cerbos vs OPA（2025/11 更新）: https://www.cerbos.dev/blog/cerbos-vs-opa
- 外部認可プラットフォーム比較（2026/03）: https://sph.sh/en/posts/external-authorization-management-systems/
- Cedar vs Rego vs OpenFGA: https://sph.sh/en/posts/policy-language-comparison-cedar-rego-openfga/
- オープンソース認可ツール 2026: https://www.permit.io/blog/top-open-source-authorization-tools-for-enterprises-in-2026
- Cerbos vs Permit.io vs OPA: https://www.pkgpulse.com/guides/cerbos-vs-permit-vs-opa-authorization-as-a-service-2026

### 行レベルフィルタ生成

- Cerbos Query Plan Adapters ガイド: https://www.mintlify.com/cerbos/cerbos/integration/query-plan-adapters
- Cerbos PlanResources 詳説: https://www.cerbos.dev/blog/filtering-database-results-with-cerbos-query-plans
- Trino RLS with Cerbos Synapse（2026/04）: https://www.cerbos.dev/blog/row-level-security-for-apache-trino
- Cerbos + Drizzle ORM: https://www.cerbos.dev/ecosystem/drizzle
- Cerbos + Prisma: https://www.cerbos.dev/ecosystem/cerbos-prisma
- Cerbos + Convex（AST 分割の例）: https://www.cerbos.dev/blog/query-plan-adapter-for-convex
- Cerbos + ChromaDB（RAG 用）: https://www.cerbos.dev/ecosystem/chromadb

### アーキテクチャパターン

- OWASP Microservices Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html
- OWASP 認可パターン改訂提案: https://www.innoq.com/en/blog/2025/06/owasp-microservice-security-cheat-sheet-update-authorization-patterns/
- AWS Prescriptive Guidance（PEP/PDP 実装）: https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/

### AI エージェント認可

- Microsoft: AI エージェント認可 Fabric（PEP+PDP）（2026/04）: https://techcommunity.microsoft.com/blog/microsoft-security-blog/authorization-and-governance-for-ai-agents-runtime-authorization-beyond-identity/4509161
- Gartner IAM London（2026/03）: https://www.cerbos.dev/blog/authorization-main-character-at-gartner-iam-london
- 2026 OAuth/Agentic AI ガイド: https://www.strata.io/blog/agentic-identity/why-agentic-ai-demands-more-from-oauth-6a/
- OBO 認証 for AI Agents: https://www.scalekit.com/blog/delegated-agent-access
- Policy-as-Code for Enterprise AI Agents: https://petronellatech.com/blog/policy-as-code-for-enterprise-ai-agents-identity-least-privilege/

### Cedar / AWS Verified Permissions

- AWS Verified Permissions & Cedar 完全ガイド（2026/05）: https://hidekazu-konishi.com/entry/aws_verified_permissions_cedar_complete_guide.html
