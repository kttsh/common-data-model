# Authorization Boundaries and Interface Decision

> ステータス: 決定ドラフト（討議用）
> スコープ: 権限管理を独立した「権限機構（PEP/PDP）」として置く構成の責務分担とインターフェース
> 前提: 認証=Entra ID、認可属性=OpenGIM（社内ユーザーリポジトリ）、データ=BigQuery、API=FastAPI/App Service、入口=Azure API Management（既存共有）、DAB 不採用
> Related: `authorization-strategy.md`, `../04-research/authorization-models-and-standards-2026.md`, `row-level-filtering-layering.md`

---

## 略語・用語

| 略語 | 正式名 | 説明 |
|---|---|---|
| PEP | Policy Enforcement Point | ポリシー実施点（実際に許可/拒否を行う箇所＝FastAPI 側） |
| PDP | Policy Decision Point | ポリシー判定点（ポリシーを評価して可否を返す箇所＝権限機構） |
| ABAC | Attribute-Based Access Control | 属性ベースアクセス制御 |
| RLS | Row-Level Security | 行レベルセキュリティ |
| CLS | Column-Level Security | 列レベルセキュリティ |
| APIM | Azure API Management | Azure の API ゲートウェイ |
| SA | Service Account | サービスアカウント |
| JWT | JSON Web Token | 署名付きトークン |
| OBO | On-Behalf-Of | 代理（人の文脈を引き継いで実行） |
| RFC | Request for Comments | IETF の標準仕様文書（例：RFC 8693 ＝ OAuth 2.0 Token Exchange） |
| AuthZEN | OpenID AuthZEN | OpenID Foundation の認可標準（Authorization API 1.0、PEP↔PDP 通信を標準化） |
| OPA | Open Policy Agent | Rego を使うポリシーエンジン |
| Rego | （固有名） | OPA のポリシー記述言語 |
| CEL | Common Expression Language | Cerbos 等が条件記述に使う式言語 |
| Cerbos | （製品名） | 外部 PDP 型の認可エンジン |
| AST | Abstract Syntax Tree | 抽象構文木（条件を木構造で表現。WHERE へ翻訳） |
| LRU | Least Recently Used | 古いものから捨てるキャッシュ方式 |
| TTL | Time To Live | キャッシュ等の有効期間 |
| DAB | Data API Builder | Microsoft の API 自動生成ツール（本件は不採用） |

---

## 1. 決定サマリ

| 項目 | 決定 |
|---|---|
| 認可の置き方 | 権限を **独立した機構（PEP/PDP）** として FastAPI と分離。ハンドラに散らさない |
| 「裁き」の構造 | **2段**：①粗い gate（モデル/エンドポイント単位の allow/deny）→ ②行・列フィルタ生成 |
| 機構の物理形態 | **まず A：FastAPI 内ライブラリ/モジュール（埋め込み PEP+PDP）** で開始。将来 B：別 PDP サービスへ |
| 機構と APIM の境界 | APIM＝入口の粗い検証、機構＝App Service 側で属性取得・判断・フィルタ生成 |
| インターフェース | `authorize(subject, action, resource) -> Decision`。Decision は allow/deny だけでなく **行フィルタ・列マスク指定** を返す |
| 行絞り込みの執行 | 機構が返したフィルタを FastAPI が **パラメータ化 WHERE** に翻訳し、BigQuery へ押し下げ（pre-filter） |
| 標準準拠 | PEP↔PDP は AuthZEN 1.0 を意識（後でエンジン差し替え可能に）。SA/エージェントは RFC 8693 の delegation/impersonation で区別 |

---

## 2. 構成と責務分担

```
[利用者：人間 / SA・AIエージェント]
        │ JWT
        ▼
[Azure API Management]  ── 入口の粗い検証（JWT 事前検証・スロットル・サブスク）
        │
        ▼
[権限機構（PEP / PDP）]  ── 判断：本人特定 → OpenGIM 属性取得 → ABAC 評価
        │                     ├ ① 粗い gate（deny なら 403）
        │                     └ ② 行フィルタ ＋ 列マスク指定を生成
        │  ← (属性参照) ── [OpenGIM：社内ユーザーリポジトリ]
        │ allow ＋ フィルタ・マスク
        ▼
[FastAPI ハンドラ]      ── 執行：フィルタ→パラメータ化 WHERE 翻訳、列マスク適用、監査記録
        │ パラメータ化 SQL
        ▼
[BigQuery]             ── 認可ビュー＋押し下げ実行（pre-filter、パーティション/クラスタで安く）
```

| 層 | 責務 | やらないこと |
|---|---|---|
| Azure API Management | JWT 事前検証、スロットリング、サブスクリプション、ルーティング、短期キャッシュ | 行・列の絞り込み（責務外） |
| 権限機構（PEP/PDP） | OpenGIM 属性取得、ABAC 評価、①粗い gate、②行フィルタ・列マスク指定の生成、判定理由の提供 | SQL の組み立て・実行 |
| FastAPI ハンドラ | フィルタを BigQuery のパラメータ化 WHERE へ翻訳、列マスク適用、監査ログ記録 | 認可ルールそのものの保持（機構に集約） |
| BigQuery | 押し下げ WHERE の実行、認可ビュー/認可データセットで共通モデル公開・SA 最小権限、パーティション/クラスタ、CLS（多層防御） | per-user の行判断（単一 SA では不可） |

---

## 3. 「裁き」を2段に分ける（重要）

権限機構が行う「裁き」は性質の違う2つに分かれる。これを混同しないことが設計の肝。

- **① 粗い gate（事前に完結できる）**：このユーザーがこのモデル/エンドポイントに触れてよいか。boolean の allow/deny。deny なら FastAPI のデータ処理に入る前に **403 で短絡**できる。
- **② 行・列フィルタ（事前に完結できない）**：どの行・どの列まで見えるか。これは BigQuery クエリに WHERE/マスクとして織り込む必要があり、クエリを組み立てるのは FastAPI の中。したがって機構が返すのは pass/fail ではなく **フィルタ（クエリプラン）＋マスク指定**で、それを FastAPI が実行時に適用する。

> 言い換え：「前段で裁き切って FastAPI に丸投げ」ではなく、「**機構が判断とフィルタを返し、FastAPI が執行する**」分業。Cerbos PlanResources / OPA Compile が allow/deny ではなくフィルタ AST を返すのはこのため。

---

## 4. 機構の物理形態（3案と決定）

| 案 | 形態 | 長所 | 短所 | 制約適合 |
|---|---|---|---|---|
| **A（採用：起点）** | FastAPI 内ライブラリ/モジュール（埋め込み PEP+PDP） | プロセス内・低レイテンシ・追加サービス承認不要 | ポリシーが各アプリに分散しがち | App Service/無コンテナ制約に最適 |
| B（将来） | 別 PDP サービス（Cerbos/OPA を別 App Service・Function でホスト、HTTP で呼ぶ） | 中央チームがポリシーを一元管理・監査、AuthZEN で差し替え可 | ホップ1回分のレイテンシ、ホスト追加 | サイドカー不可のため別ホスト＋短 TTL キャッシュ |
| C（不採用） | FastAPI の「前」に立つプロキシで裁いてから転送 | ①粗い gate は可 | ②行・列は不可（リクエスト書き換えは脆弱）、APIM と機能重複 | APIM が既に入口を担うため重複 |

**決定**：まず **A（埋め込み）** で開始。ポリシーは宣言的（CEL / Rego 等）に書き、共通リポジトリ/パッケージとして配布して分散を防ぐ。モデル API が増え中央集権ガバナンスを強める段階で **B（別 PDP）** へ移行。AuthZEN 準拠のインターフェースにしておけば、移行時に PEP 側（FastAPI）コードの変更を最小化できる。

---

## 5. authorize() インターフェース案

機構は FastAPI から1関数で呼べる PEP/PDP として設計する。allow/deny だけでなく、**行フィルタと列マスク指定**を返すのが要点。

```
authorize(
  subject,    # 認証済みプリンシパル
              #   人間: Entra ID の oid/sub ＋ OpenGIM 突合キー
              #   SA/エージェント: RFC 8693 の act/may_act で「実行主体」と「代理元」を区別
  action,     # "read" など（GET 系は read）
  resource    # 対象。例: { model: "部門費", kind: "record" }
) -> Decision

Decision = {
  effect: "allow" | "deny",
  reason: str,                  # 監査用（なぜ許可/拒否したか）
  row_filter: <条件AST> | None,  # 行レベル：パラメータ化 WHERE へ翻訳する条件木
  masked_columns: [str],        # 列レベル：整形時にマスクする列
  obligations: { ... }          # 任意：ステップアップ要求・追加監査タグ等
}
```

属性取得（OpenGIM 参照）は **機構の内部**で行い、subject のキーで突合する。FastAPI 側は属性の存在を意識しない。

### FastAPI からの呼び出し例（依存性として）

```python
@router.get("/models/{model}/records")
async def get_records(model: str, principal = Depends(authenticate)):
    decision = authz.authorize(principal, "read", Resource(model=model))
    if decision.effect == "deny":
        raise HTTPException(status_code=403, detail=decision.reason)   # ① 粗い gate

    sql, params = compile_to_bigquery(base_query(model), decision.row_filter)  # ② フィルタ→WHERE
    rows = await bq.query(sql, params)                # pre-filter（BigQuery に押し下げ）
    result = apply_mask(rows, decision.masked_columns)  # 列マスク

    audit.log(principal, model, decision.reason, applied_filter=sql)  # 監査：適用条件を記録
    return result
```

- `compile_to_bigquery` が「条件 AST → パラメータ化 BigQuery SQL」の翻訳層（BigQuery 向けは自作）。フィルタ列はパーティション/クラスタに合わせ、関数で包まず直接比較にしてプルーニングを効かせる。
- ユーザー由来値は文字列連結せず BigQuery のクエリパラメータ（`@param` / `UNNEST(@array)`）で渡す。

---

## 6. APIM との境界（重複回避）

- **APIM** = 入口の粗い検証（JWT 事前検証・スロットル・サブスク・ルーティング）。
- **権限機構** = App Service 側で OpenGIM 属性取得 → ABAC → ①gate → ②フィルタ/マスク生成。
- C 案（前段プロキシで裁く）を自作すると APIM と機能が重複するため採らない。`../02-architecture/platform-architecture-decision.md` の「APIM で JWT 事前検証 → App Service で再検証＋属性ベース ABAC」の二段と整合。

---

## 7. 監査・キャッシュの注意

- **監査**：機構は RLS のように「黙って絞る」挙動を担うため、`Decision.reason` と **実際に適用したフィルタ条件**（生成 WHERE）を監査ログに必ず残す。
- **キャッシュ**：per-user のフィルタが付く結果は共有キャッシュのヒット率が低い。共通参照系（マスタ・スキーマ・公開メタ）に限定し、属性（OpenGIM）はプロセス内 LRU で短 TTL。PDP を B 化したら判定結果も短 TTL でキャッシュしてホップを緩和。

---

## 8. 未決事項 / 次アクション

| # | 項目 | 内容 |
|---|---|---|
| 1 | ポリシー言語/エンジン | 埋め込み（PyCasbin / 自前 + CEL）か、外部 PDP（Cerbos/OPA）かの最終比較（D2 と連動） |
| 2 | 条件 AST → BigQuery 翻訳層 | PlanResources/Compile 出力 → BigQuery パラメータ化 SQL のマッピング設計 |
| 3 | SA/エージェントの token | RFC 8693 delegation（OBO）/ impersonation の使い分けと Entra ID 実装（OBO フロー）確認 |
| 4 | OpenGIM 参照仕様 | 突合キー・属性項目・到達経路（論点② と連動） |
| 5 | PoC | 「部門費モデル」で subject→属性→Decision→生成 WHERE→BigQuery 実行→列マスク の一連を最小実装 |
