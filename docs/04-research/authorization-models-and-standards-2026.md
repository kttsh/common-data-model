# Authorization Models, Standards, and FastAPI Practices (2026)

> ステータス: 調査メモ（世の中のプラクティス）
> Purpose: Research backing for `../03-authorization/authorization-strategy.md`, especially SQL governance, OBO/service identity, and audit logging.
> 前提: 認証=Entra ID、認可属性=Open-GIM（社内ユーザーリポジトリ）、データ=BigQuery、実行基盤=FastAPI + App Service
> 情報時点: 2026年5月

---

## 略語・用語

| 略語 | 正式名 | 説明 |
|---|---|---|
| RBAC | Role-Based Access Control | ロールベースアクセス制御 |
| ABAC | Attribute-Based Access Control | 属性ベースアクセス制御 |
| ReBAC | Relationship-Based Access Control | 関係ベースアクセス制御 |
| PBAC | Policy-Based Access Control | ポリシーベースアクセス制御 |
| ACL | Access Control List | アクセス制御リスト |
| DAC | Discretionary Access Control | 任意アクセス制御 |
| PEP | Policy Enforcement Point | ポリシー実施点（実際に許可/拒否を行う箇所） |
| PDP | Policy Decision Point | ポリシー判定点（ポリシーを評価して可否を返す箇所） |
| PIP | Policy Information Point | ポリシー情報点（判定に使う属性を提供する箇所） |
| PAP | Policy Administration Point | ポリシー管理点（ポリシーを作成・管理する箇所） |
| XACML | eXtensible Access Control Markup Language | OASIS の ABAC 標準 |
| ALFA | Abbreviated Language For Authorization | XACML を書きやすくした略記言語 |
| AuthZEN | OpenID AuthZEN | OpenID Foundation の認可標準（Authorization API 1.0） |
| OIDC | OpenID Connect | OAuth 2.0 上の認証レイヤ |
| JWT | JSON Web Token | 署名付きトークン |
| OBO | On-Behalf-Of | 代理（人の文脈を引き継いで実行） |
| RFC | Request for Comments | IETF の標準仕様文書 |
| SCIM | System for Cross-domain Identity Management | ID プロビジョニング標準 |
| OPA | Open Policy Agent | Rego を使うポリシーエンジン |
| Rego | （固有名） | OPA のポリシー記述言語 |
| CEL | Common Expression Language | Cerbos 等が条件記述に使う式言語 |
| Cerbos / Cedar | （製品名） | 認可エンジン／ポリシー言語 |
| AST | Abstract Syntax Tree | 抽象構文木（条件を木構造で表現） |
| SA | Service Account | サービスアカウント |
| TTL | Time To Live | キャッシュ等の有効期間 |

---

## 1. 認可モデルの整理（RBAC / ABAC / ReBAC / PBAC ほか）

「他にもある？」への答えとして、実務で名前が挙がる主なモデルを並べる。

| モデル | 判定の軸 | 得意なこと | 苦手なこと |
|---|---|---|---|
| **RBAC**（ロールベース） | ユーザーのロール | 単純・運用が軽い。粗い入口制御 | 条件が増えるとロール爆発。動的条件に弱い |
| **ABAC**（属性ベース） | 主体・リソース・環境（時刻/場所等）の属性 | 行・列の動的制御、実行時の文脈判断、細粒度 | ポリシーが複雑化しやすい。設計が要る |
| **ReBAC**（関係ベース） | 主体とリソースの関係グラフ | 「所有者」「所属ツリー」など関係・階層、巨大規模 | 時刻・場所など動的属性の細かい条件は苦手 |
| **PBAC**（ポリシーベース） | 明示的・監査可能なポリシー（RBAC/ABAC を内包） | ルールを一箇所に集約、テスト・監査・可観測性 | ポリシーエンジン導入の前提が要る |
| （参考）ACL / DAC | リソースごとの許可リスト／所有者裁量 | 小規模・単純 | 大規模・横断ガバナンスに不向き |

**2026 年の主流の捉え方**

- 「RBAC か ABAC か」ではなく、**組み合わせて使う**のが定石。多くの本番系は RBAC（粗い）+ ReBAC（リソース単位）を基本にし、ABAC が両者でカバーできない条件付き判断（時刻・場所・センシティブ度など）を埋める。
- PBAC は RBAC/ABAC を置き換えるのではなく、それらを**ポリシーとして統合**する考え方。Oso / OPA のようなポリシーエンジンがこの層を担う。
- 「誰が・何を・どの条件で」は、もはや単純な RBAC テーブルではなく「常に変化するグラフ」として捉える、というのが各社の論調。

---

## 2. 今回の要件への適合性

要件（role memo / `docs/01-requirements/product-requirements.md`）を各モデルに割り付ける。

| 今回の要件 | 最適なモデル | 補足 |
|---|---|---|
| 行レベル：自部門のみ／主席・部長以上は全社、など属性しきい値依存 | **ABAC** | 役職レベル・所属・居住地などの属性で動的に WHERE を決める典型例 |
| 行レベル：起票者単位（自分が起票したレコードのみ） | ABAC でも ReBAC でも可 | `@item.creator == subject.id` で ABAC 表現可。関係が増えるなら ReBAC 部分採用の余地 |
| 列レベル：原価・単価・個人情報のマスク | **ABAC** | 属性に応じて返す列／マスクする列を決める |
| 異動時に自動追従（棚卸不要） | **ABAC ＋ 最新属性ソース** | 属性が最新（Open-GIM が最新）なら判定も自動追従。鮮度は属性ソース依存 |
| モデル単位の入口制御（このモデルは全社可／人事のみ） | RBAC 的 ／ ABAC | 「モデルごとのデフォルト権限」は粗い RBAC 層として表現できる |

**結論（モデル選定の叩き）**

- **中核は ABAC**。役職・所属・居住地・原価センタといった属性で行・列を出し分ける今回の要件と素直に噛み合う。
- **入口は RBAC 的なデフォルト層**（モデル単位の「全社可／部門限定」）を ABAC ポリシーの一部として表現。
- **ReBAC は将来オプション**。起票者単位・組織ツリーなど「関係」が支配的になってきたら部分採用を検討（ABAC だけでは関係グラフの巡回が重くなるため）。
- いずれにせよ、ルールは**ポリシーとして外出し**（自由 SQL ではなく宣言的ポリシー）にするのが PBAC 的な定石で、論点③（自由 SQL の統制）への回答になる。

---

## 3. 標準・RFC（仕様として決まっているもの）

「RFC 他標準で決まっているもの」を、認証/トークン系と認可系に分けて整理。

### 3.1 認証・トークン系（主に IETF / OpenID）

| 標準 | 内容 | 今回の関わり |
|---|---|---|
| OAuth 2.0（RFC 6749） | 認可フレームワーク | Entra ID の土台 |
| Bearer Token Usage（RFC 6750） | `Authorization: Bearer` の使い方 | API のトークン受け渡し |
| JWT（RFC 7519） | JSON Web Token の構造 | アクセストークン本体 |
| JWT Profile for OAuth 2.0 Access Tokens（RFC 9068） | アクセストークンを JWT として標準化（`sub`/`aud`/`scope`/`roles` 等の扱い） | App Service 側 JWT 検証の準拠先 |
| JWT Best Current Practices（RFC 8725 / BCP 225） | JWT の安全な扱い | `alg` 固定・`aud`/`iss` 検証など実装規約 |
| OAuth 2.0 Token Exchange（RFC 8693） | トークン交換。**impersonation（なりすまし）と delegation（委譲）を区別**。`subject_token`＝代理される側、`actor_token`＝実行する側、`act`/`may_act` クレームで合成トークンを表現 | **論点⑤（T ユーザー／AI エージェント）の標準的根拠**。Azure の On-Behalf-Of（OBO）はこのパターンの実装 |
| Resource Indicators（RFC 8707） | トークンの想定リソースを示す | マルチ API 時の audience 制御 |
| OpenID Connect（OIDC） | OAuth2 上の認証レイヤ | Entra ID の認証 |
| SCIM（RFC 7643 / 7644） | ID・属性のプロビジョニング | 将来 SA／属性連携を標準化する余地 |

> ポイント：role memo の「システムは T ユーザー」と「AI エージェントが人の所属で絞られる」は、RFC 8693 の用語で言うと前者が **impersonation/サービス ID**、後者が **delegation（OBO）**。両者を分けて定義すれば曖昧さが消える。

### 3.2 認可系（ABAC / ポリシー）

| 標準 | 内容 | 今回の関わり |
|---|---|---|
| **XACML**（OASIS） | ABAC の老舗標準。PEP（実施点）/ PDP（判定点）/ PIP（属性提供）/ PAP（管理）の役割分担を定義 | 認可アーキテクチャの語彙の出所。重厚で XML ベース |
| **NIST SP 800-162** | ABAC のリファレンスガイド。PEP-PDP パターンを規定 | ABAC を採る際の設計の典拠 |
| **OpenID AuthZEN Authorization API 1.0** | **2026 年に OpenID Foundation の Standards Track として公開**。PEP↔PDP の通信を JSON API で標準化。情報モデルは **4 つ組（Subject / Action / Resource / Context）**、判定は基本 boolean。**ポリシー言語には非依存**（OPA / Cedar / XACML / Topaz などをそのまま PDP として使える）。応答にマスク指示・監査メタデータ・ステップアップ要求等を載せられる | PEP（＝FastAPI）と PDP を疎結合にし、後からエンジンを差し替え可能にする。「認可の OIDC」と呼ばれる |
| **Google Zanzibar（論文）** | ReBAC の事実上の参照設計。実装は OpenFGA / SpiceDB / Permify / Ory Keto など | 将来 ReBAC を採るときの選択肢 |
| ALFA | XACML を書きやすくした略記言語 | XACML 系を採るなら |

> 補足：AuthZEN は「ポリシーの書き方」は決めず「PEP と PDP の会話の仕方」だけを決める標準。だから OPA でも Cedar でも、後で乗り換えても PEP 側のコードを書き換えずに済む。今回のように「まず作って後で育てる」方針と相性が良い。

---

## 4. FastAPI 実装に適したライブラリ

FastAPI 自身は認可機構を持たない（公式も「専用ライブラリを使え」という立場）。今回のような「JWT 検証 → 外部属性参照 → 行・列フィルタ → 監査」を組むのに使える選択肢を整理。

### 4.1 JWT 検証（認証の出口）

- **PyJWT** / **Authlib**：JWKS 取得・`aud`/`iss`/`exp` 検証。軽量。
- **msal**（Microsoft 純正）：Entra ID 連携が素直。OBO（RFC 8693 系）もサポート。

### 4.2 認可エンジン／ポリシー

| ライブラリ | 形態 | モデル | 行レベルフィルタ生成 | メモ |
|---|---|---|---|---|
| **Casbin（PyCasbin）** | 埋め込み（プロセス内） | ACL/RBAC/ABAC/ReBAC ほか（PERM メタモデル） | △（マッチャで属性判定は可、WHERE 生成は弱い） | `fastapi-casbin-auth`（ミドルウェア）/ `casbin-fastapi-decorator`（デコレータ）。多言語同一 API。低レイテンシ |
| **Cerbos** | 外部 PDP（別プロセス/サービス） | ABAC/RBAC（派生ロール）、CEL 条件、YAML ポリシー | **◎ PlanResources（Query Plan）**：ポリシーを部分評価して AST を返し、DB フィルタに翻訳。**Python は SQLAlchemy アダプタあり** | 行フィルタも列出力（マスク指示）もポリシー側で記述可。ステートレス |
| **OPA（Open Policy Agent）** | 外部 PDP（別プロセス/サービス） | Rego で何でも | **◎ Compile API（部分評価）**：未知変数に依存する残余ポリシーを返し、SQL 述語に翻訳可 | CNCF。AuthZEN 1.0 の PDP として標準対応の動きあり |
| **Cedar（AWS）** | ライブラリ／マネージド（Verified Permissions） | RBAC/ABAC | 部分評価あり | 検証（formal verification）に強い独自言語。マネージドは AWS 寄りで閉域 Azure とは相性に注意 |
| **Oso** | ライブラリ／Oso Cloud | RBAC/ABAC/ReBAC（Polar 言語） | データフィルタ機能あり | 近年は **Oso Cloud（マネージド）中心**に軸足。自己ホスト前提なら Casbin/Cerbos の方が素直 |
| **py-abac / Vakt** | 埋め込み | ABAC（XACML 風） | △ | 純 Python の ABAC。小規模・自己完結向け |
| **OpenFGA / SpiceDB** | 外部サービス | ReBAC（Zanzibar） | 関係クエリ | 将来 ReBAC を採るとき |

### 4.3 今回の肝：「行レベル＝ポリシーからフィルタを生成」パターン

role memo の「SQL ライクに WHERE を設定して注入」は、**自由 SQL ではなく、ポリシーから機械的にフィルタを生成する**形にするのが 2026 年の定石。代表的に 2 方式：

- **Cerbos PlanResources（Query Plan）**：特定の 1 レコードではなく「このユーザーがこのアクションで見られる条件」を部分評価し、抽象構文木（AST）として返す。これを DB のフィルタ言語に翻訳して**データ層に絞り込みを押し込む**（アプリで全行取得して捨てない）。Python では SQLAlchemy アダプタが提供されている。
- **OPA Compile API（部分評価）**：Rego ポリシーのうち実行時にしか分からない部分を残余 AST として返し、SQL 述語などに変換する。

**今回特有の注意：BigQuery 用アダプタは自作になる**
Cerbos/OPA のアダプタは Prisma / SQLAlchemy / Elasticsearch 等が中心で、**BigQuery 向けの既製アダプタは無い**。返ってくるのはクエリ言語非依存の AST なので、**AST → BigQuery SQL（パラメータ化クエリ）への翻訳層を 1 つ自作**すれば成立する。これにより「オーナーが書くのは宣言的ポリシー、SQL を組むのはアダプタ」という分離ができ、論点③（自由 SQL のインジェクション・検証・レビュー問題）を構造的に解消できる。

**列レベル（マスク）**
列制御は「どの列を返す／マスクするか」をポリシー判定の出力として返し、API のレスポンス整形時に適用するのが素直（`docs/02-architecture/platform-architecture-decision.md` の「列レベル＝整形時マスク」と一致）。AuthZEN もマスク指示を応答に載せる設計を許容している。

---

## 5. デプロイ観点の注意（App Service／無コンテナ制約との整合）

PDP の置き方は、`docs/02-architecture/platform-architecture-decision.md` の制約（バックエンドは App Service / Functions のみ、Container Apps / AKS 不可）と擦り合わせる必要がある。

| 方式 | レイテンシ | ガバナンス／フィルタ生成 | App Service との相性 |
|---|---|---|---|
| **埋め込み（PyCasbin / py-abac）** | ◎ プロセス内、追加サービス無し | △ WHERE 生成は弱い | ◎ そのまま動く |
| **外部 PDP（Cerbos / OPA）** | ○ ネットワークホップ 1 回（短 TTL でキャッシュ） | ◎ 部分評価・ポリシー集中管理・監査 | △ サイドカーは App Service ネイティブ非対応。PDP を別 App Service / Function でホストするか、埋め込み/ePDP を検討 |

- 外部 PDP は「ポリシーを一箇所で管理・テスト・監査できる」「行フィルタを生成できる」点が強いが、本構成ではサイドカー同梱が難しい。**PDP を別ホストに置いてホップを許容**するか、**判定結果を短 TTL でキャッシュ**して 100〜500ms 目標を守るのが現実解。
- **AuthZEN 準拠の PEP として FastAPI を書いておく**と、最初は埋め込み、後から外部 PDP、という乗り換えが PEP 側コード変更なしでできる。

---

## 6. 推奨の方向性（叩き）

1. **モデル**：ABAC を中核に、モデル単位のデフォルト（粗い RBAC 層）と監査を組み合わせる。起票者・組織ツリーが効いてきたら ReBAC を部分採用。
2. **実装**：FastAPI を PEP として実装。ルールは**宣言的ポリシー（CEL / Rego 等）で外出し**し、**自由 SQL は禁止**。行レベルは PlanResources / Compile 方式の「ポリシー→フィルタ生成」を **BigQuery 向け翻訳層**経由で適用。列レベルはポリシー出力でマスク指定。
3. **標準準拠**：トークンは RFC 9068 準拠の JWT を RFC 8725（BCP）に沿って検証。SA／AI エージェントは RFC 8693 の **delegation（OBO）** で人の文脈を引き継ぎ、サービス固定実行は impersonation/サービス ID として分離。PEP↔PDP は **AuthZEN 1.0** を意識して疎結合に。
4. **監査**：判定で実際に適用したフィルタ条件（生成 WHERE）と判定理由をログに残す（論点⑧）。

> この方向性は、別ファイルの論点③（自由 SQL の統制）・⑤（T ユーザー／OBO）・⑧（監査の条件記録）に直接対応する。

---

## 参考（出典）

- RBAC vs ABAC vs PBAC（PBAC が RBAC/ABAC を内包・ReBAC へ拡張）: https://www.osohq.com/learn/rbac-vs-abac-vs-pbac
- RBAC vs ABAC vs ReBAC（2026、RBAC+ReBAC 合成が主流、Zanzibar 実装一覧）: https://guptadeepak.com/ciam-compass/guides/rbac-vs-abac-vs-rebac/
- RBAC vs ABAC（ABAC の行・列/マスク/時間帯制御の例）: https://www.strongdm.com/blog/rbac-vs-abac
- RBAC vs ABAC vs ReBAC（モデル併用の考え方）: https://www.permit.io/blog/rbac-vs-abac-vs-rebac
- OpenID AuthZEN 仕様サイト（PEP-PDP、Authorization API 1.0）: https://openid.github.io/authzen/
- AuthZEN 1.0 ディープダイブ（4 つ組情報モデル、ポリシー非依存、2026 公開）: https://dev.to/kanywst/authzen-authorization-api-10-deep-dive-the-standard-api-that-separates-authorization-decisions-1m2a
- AuthZEN 概要（XACML/OPA を跨ぐ相互運用、policy-agnostic）: https://nordicapis.com/authzen-a-new-standard-for-fine-grained-authorization/
- AuthZEN と NIST ABAC 800-162 / XACML / ALFA: https://axiomatics.com/resources/reference-library/openid-authzen
- AuthZEN を FastAPI から PEP として呼ぶ実装ガイド（短 TTL キャッシュ）: https://auth0.com/blog/implementing-authzen-guide-openid-authorization-api/
- OPA に AuthZEN 1.0 対応（PDP として /access/v1/evaluation）: https://github.com/open-policy-agent/opa/issues/8449
- オープンソース認可ツール 2026（Casbin/OPA/Cedar/OPAL ほか）: https://www.permit.io/blog/top-open-source-authorization-tools-for-enterprises-in-2026
- Apache Casbin 公式（PERM メタモデル、対応モデル）: https://casbin.apache.org/
- fastapi-casbin-auth（PyPI）: https://pypi.org/project/fastapi-casbin-auth/
- Casbin デコレータ統合（ABAC/所有権ベース）: https://dev.to/neko1313/fastapi-authorization-without-middleware-decorator-based-casbin-integration-4n3o
- FastAPI 公式ディスカッション（認可は専用ライブラリ＝Oso/Cerbos/Casbin 等）: https://github.com/fastapi/fastapi/discussions/8413
- Cerbos PlanResources / Query Plan（部分評価→AST→DB フィルタ）: https://www.cerbos.dev/blog/filtering-database-results-with-cerbos-query-plans
- Cerbos Query Plan アダプタ（Python SQLAlchemy 対応、5 分キャッシュ例）: https://www.mintlify.com/cerbos/cerbos/integration/query-plan-adapters
- Cerbos で行フィルタ＋列出力（SQL WHERE を返す例）: https://www.cerbos.dev/blog/row-level-security-for-apache-trino
- OPA Compile API による SQL データフィルタリング: https://github.com/open-policy-agent/opa/issues/830
- OAuth 2.0 Token Exchange（RFC 8693、impersonation/delegation、act/may_act）: https://www.rfc-editor.org/info/rfc8693/
- JWT Profile for OAuth 2.0 Access Tokens（RFC 9068）: https://www.rfc-editor.org/rfc/rfc9068.html
- RFC 8693 解説（impersonation と delegation の違い）: https://dev.to/kanywst/rfc-8693-deep-dive-token-exchange-310i
