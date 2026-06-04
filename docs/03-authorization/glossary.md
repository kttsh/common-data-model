# 認可ドキュメント 用語集

認可関連ドキュメント（`03-authorization/` および `04-research/` の認可系）で使う略語・用語の**正本**。各ドキュメントは個別の略語表を持たず、本ファイルを参照する。新しい用語が増えたらここだけを更新する。

---

## 認可モデル

| 略語 | 正式名 | 説明 |
|---|---|---|
| RBAC | Role-Based Access Control | ロールベースアクセス制御 |
| ABAC | Attribute-Based Access Control | 属性ベースアクセス制御 |
| ReBAC | Relationship-Based Access Control | 関係ベースアクセス制御 |
| PBAC | Policy-Based Access Control | ポリシーベースアクセス制御（RBAC/ABAC を内包） |
| ACL | Access Control List | アクセス制御リスト |
| DAC | Discretionary Access Control | 任意アクセス制御 |

## PEP/PDP 役割（XACML 由来）

| 略語 | 正式名 | 説明 |
|---|---|---|
| PEP | Policy Enforcement Point | ポリシー実施点（実際に許可/拒否を行う箇所＝FastAPI 側） |
| PDP | Policy Decision Point | ポリシー判定点（ポリシーを評価して可否を返す箇所＝権限機構） |
| PIP | Policy Information Point | ポリシー情報点（判定に使う属性を提供する箇所＝Open-GIM 等） |
| PAP | Policy Administration Point | ポリシー管理点（ポリシーを作成・管理する箇所） |

## 行・列制御

| 略語 | 正式名 | 説明 |
|---|---|---|
| RLS | Row-Level Security | 行レベルセキュリティ |
| CLS | Column-Level Security | 列レベルセキュリティ |
| AST | Abstract Syntax Tree | 抽象構文木（条件を木構造で表現。WHERE へ翻訳） |
| MV | Materialized View | マテリアライズドビュー |

## 認証・トークン

| 略語 | 正式名 | 説明 |
|---|---|---|
| JWT | JSON Web Token | 署名付きトークン |
| OIDC | OpenID Connect | OAuth 2.0 上の認証レイヤ |
| OBO | On-Behalf-Of | 代理（人の文脈を引き継いで実行） |
| RFC | Request for Comments | IETF の標準仕様文書（例：RFC 8693 ＝ OAuth 2.0 Token Exchange） |
| SCIM | System for Cross-domain Identity Management | ID・属性のプロビジョニング標準 |

## 標準・仕様

| 略語 | 正式名 | 説明 |
|---|---|---|
| XACML | eXtensible Access Control Markup Language | OASIS の ABAC 標準 |
| ALFA | Abbreviated Language For Authorization | XACML を書きやすくした略記言語 |
| AuthZEN | OpenID AuthZEN | OpenID Foundation の認可標準（Authorization API 1.0、PEP↔PDP 通信を標準化） |

## ポリシーエンジン・言語

| 略語 | 正式名 | 説明 |
|---|---|---|
| OPA | Open Policy Agent | Rego を使うポリシーエンジン |
| Rego | （固有名） | OPA のポリシー記述言語 |
| CEL | Common Expression Language | Cerbos 等が条件記述に使う式言語 |
| Cerbos | （製品名） | 外部 PDP 型の認可エンジン |
| Cedar | （製品名） | AWS 由来のポリシー言語／エンジン |
| OpenFGA | （製品名） | Zanzibar 系 ReBAC エンジン |

## 基盤・運用

| 略語 | 正式名 | 説明 |
|---|---|---|
| APIM | Azure API Management | Azure の API ゲートウェイ |
| SA | Service Account | サービスアカウント |
| IAM | Identity and Access Management | ID・アクセス管理 |
| LRU | Least Recently Used | 古いものから捨てるキャッシュ方式 |
| TTL | Time To Live | キャッシュ等の有効期間 |
| SLO | Service Level Objective | サービスレベル目標 |
| DAB | Data API Builder | Microsoft の API 自動生成ツール（本件は不採用 → `../02-architecture/data-api-builder-assessment.md`） |
