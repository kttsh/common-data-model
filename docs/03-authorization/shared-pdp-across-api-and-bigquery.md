# Sharing the PDP Across FastAPI and BigQuery Paths

> ステータス: 調査メモ（将来オプションの検討材料）
> スコープ: PDP（ポリシー判定）を **API 経路（FastAPI）と BigQuery を叩く処理で共用する**ための手段
> 前提: 認証=Entra ID、認可属性=Open-GIM、データ=BigQuery、API=FastAPI（ARO 上のコンテナ）、API→BigQuery は単一 SA、認可は API 層 ABAC を正本
> **重要（整合）**: 認可エンジンは **PyCasbin（埋め込み）で確定済み**（`../90-research/abac-authz-library-comparison.md`、`pycasbin/`）。本メモが第一候補とする **外部 PDP（Cerbos/OPA）は将来オプション**であり、現時点の採用ではない。本メモは「PDP を複数経路で共用する」要件が確定したときに備えた検討材料として読むこと。
> Related: `row-level-filtering-layering.md`, `authorization-boundaries-and-interface.md`, `open-issues-and-options.md`（未決論点の正本）, `../90-research/authorization-models-and-standards-2026.md`
> 情報時点: 2026年6月

---

## 略語・用語

略語・用語は [`glossary.md`](glossary.md) を参照（認可ドキュメント共通の正本）。

---

## 0. 結論（先に）

- 「PDP を BigQuery と FastAPI で共用する」とは、**BigQuery エンジン自身が PDP を呼ぶことではない**。BigQuery には「クエリ実行時に外部 PDP へ問い合わせて行を絞る」機構が無い。正しくは **BigQuery を叩く“処理”（FastAPI のハンドラでも抽出バッチでも）が、クエリ発行前に同じ PDP を呼び、返ってきた条件を WHERE に変換してから叩く**こと。共用しているのは「BigQuery を叩く側のコード」。
- 共用すべきは PDP 本体だけではない。**①PDP（判定エンジン）／②認可クライアント／③AST→BigQuery SQL 翻訳層** の 3 点セットを共有ライブラリとして切り出して初めて、経路間で挙動がズレない「真の共用」になる。
- 現行の採用は **PyCasbin（埋め込み）**。**経路間共用を主目的に格上げする場合の将来オプション**として言えば、**外部 PDP（Cerbos / OPA）** が最も素直（ポリシー更新が 1 か所で済む）。**ARO はサイドカーに対応する**ため、その場合は **PDP を同一 Pod のサイドカーとして同居**させれば localhost 呼び出しでホップを最小化でき（別ホストにする場合は 1 ホップを短 TTL キャッシュで吸収）、これが現実解。埋め込み PyCasbin のままでも、後述②③の共有ライブラリ化で経路間の挙動ズレは抑えられる。
- この方式が成立する前提は **「利用者が素の BigQuery 直接接続をしない」**こと。利用者アプリが自前で BigQuery に DB 接続する要件があるなら、それは本メモ（①）では満たせず、別途「ポリシーから RLS/CLS を生成・同期する」方式（②）の検討に移る。

---

## 1. 「PDP 共用」の 2 つの意味（混同注意）

「PDP を共用する」は実は別物の 2 つを指しうる。設計を誤らないために最初に切り分ける。

| 意味 | 内容 | 本メモの扱い |
|---|---|---|
| **判定そのものを共用** | FastAPI も、BigQuery を叩く別処理も、実行時に **同じ PDP に問い合わせる** | **本命（①）**。本メモの主題 |
| **ポリシー定義だけ共用** | ポリシーは 1 か所に書くが、執行は別々（API は PDP 経由 WHERE、直接経路は別生成の RLS） | 別メモ（②）の領域。直接接続要件があるときに検討 |

本メモが扱うのは前者。鍵は **「BigQuery も FastAPI と同じトラステッド処理として PDP を呼ぶ」**という構図に揃えること。

---

## 2. なぜ BigQuery は PDP を直接呼べないか

- BigQuery ネイティブの行制御（RLS）は、**「叩いたプリンシパルの ID」＋「テーブルに静的に貼った filter_expression」**で決まる静的なもの。リクエスト毎に外部 PDP へ問い合わせて動的に絞る仕組みは持たない。
- 従って「PDP を共用」と言っても、BigQuery エンジン自身が PDP を呼ぶのではなく、**BigQuery にクエリを投げる前段のコードが PDP を呼ぶ**。
- この構図のもとでは、PDP を共用しているのは BigQuery ではなく **「BigQuery を叩く側（FastAPI ハンドラ／抽出処理）」**である。ここが腑に落ちると全体構成が見通せる。

---

## 3. 共用する 3 点セット

```
                  ┌──────────────────────────────┐
                  │  ① PDP（Cerbos / OPA）            │  判定エンジン本体
                  │     ポリシー YAML/Rego が唯一の正本   │
                  └───────────────┬──────────────┘
                                  │ PlanResources / Compile を呼ぶ
        ┌─────────────────────────┼──────────────────────────┐
        ▼                                                     ▼
  [FastAPI のハンドラ]                            [抽出バッチ / 別の BQ 利用処理]
        │                                                     │
        │  ② 共通の認可クライアント                               │  ② 同じ認可クライアント
        │     subject/action/resource を組み立てて PDP を呼ぶ      │
        │                                                     │
        │  ③ 共通の AST→BigQuery SQL 翻訳層                       │  ③ 同じ翻訳層
        │     PDP が返した条件 AST を WHERE 句に変換                │
        ▼                                                     ▼
   WHERE 付きパラメータ化クエリ                        WHERE 付きパラメータ化クエリ
        └──────────────────────┬───────────────────────────┘
                               ▼
                          [BigQuery] 認可ビュー＋パーティション／クラスタ＋CLS
```

### ① PDP 本体（判定エンジン）

Cerbos なり OPA なりを 1 インスタンス（または同一ポリシーバンドルを配る複数インスタンス）として立て、**ポリシー定義を唯一の正本**にする。同じポリシーが API レベルのアクセス制御もデータレベルのフィルタ生成も駆動し、「誰が何を見られるか」の単一の真実の源になる、というのが 2026 年の externalized authorization の標準的な狙い。

### ② 認可クライアント（共通ライブラリ）

PDP に渡す入力（subject＝誰か、action＝何をするか、resource＝どのモデルか）を組み立てて呼び出す部分。**FastAPI も抽出処理も、この同じクライアントを import して使う**。ここを共通化しないと、入力の組み立て方が経路ごとにズレ、ルールが実質二重化する。

### ③ AST→BigQuery SQL 翻訳層（共通ライブラリ）

PDP の `PlanResources`（Cerbos）や Compile API（OPA）は、判定結果として **条件の AST** を返す。これは「このユーザーがこのアクションで見てよいレコードの条件」＝実質 SQL の WHERE 句に相当し、述語をデータベースクエリに押し下げて、見てよい行だけが返るようにできる。

- **BigQuery 向けの既製アダプタは存在しない**（Cerbos/OPA のアダプタは SQLAlchemy / Prisma / Drizzle 等が中心）。よって ③ は自作。
- ただし返ってくる AST は **クエリ言語非依存**なので、翻訳層を 1 つ書けば両経路で使い回せる。
- WHERE は文字列連結で組まず、ユーザー由来の値は BigQuery のクエリパラメータ（`@param`、配列は `UNNEST(@allowed_depts)`）で渡す（`row-level-filtering-layering.md` と整合）。

---

## 4. ②③を「共有ライブラリ」に切り出すのが核心

「PDP の共用」を意味あるものにするのは、① の PDP 本体よりも **②③を共有ライブラリとして切り出すこと**。PDP を 1 つにしても、呼び出し側がバラバラに入力を作り WHERE を組んでいたら、そこで挙動がズレるため。

```
authz_core/                  ← FastAPI も抽出処理も両方が依存する単一パッケージ
├── client.py                # ② subject/action/resource を組み立てて PDP を呼ぶ
├── translate_bq.py          # ③ PDP の条件 AST → BigQuery パラメータ化 SQL
├── attributes.py            # Open-GIM から属性取得（短 TTL キャッシュ込み）
└── audit.py                 # 適用した WHERE と判定理由を監査ログへ
```

FastAPI 経路（イメージ）:

```python
decision = authz_core.client.plan(subject, action="read", resource="order_model")
where, params = authz_core.translate_bq.to_sql(decision)        # ③
rows = bq.query(f"SELECT * FROM authorized_view.orders WHERE {where}", params)
authz_core.audit.log(subject, "order_model", where)
```

抽出バッチも **同じ呼び出し**を行い、違いは結果を JSON で返すかファイルに吐くかだけ。これで「PDP を FastAPI と（BigQuery を叩く別処理が）共用する」が実体を持つ。

---

## 5. 設計判断ポイント

### (1) PDP の物理形態：埋め込みか外部か

| 方式 | 共用のしやすさ | レイテンシ | ARO との相性 |
|---|---|---|---|
| **埋め込み（PyCasbin 等／プロセス内）** | △ 別プロセス/別デプロイごとにライブラリと最新ポリシーを配る必要。配布同期が課題 | ◎ プロセス内 | ◎ そのまま動く |
| **外部 PDP（Cerbos / OPA をサイドカー / 別ホスト）** | ◎ 両経路が同じエンドポイントを叩く。ポリシー更新は 1 か所 | ◎ サイドカーなら localhost（別ホストなら 1 ホップを短 TTL で吸収） | ◎ **サイドカー可**（同一 Pod に同居）。別ホストも選択可 |

- 共用を主目的にするなら **外部 PDP が有利**。Cerbos はステートレスでサブミリ秒評価、エアギャップ/閉域環境にも対応し、サイドカー形態でも自己ホストできるため、**ARO では同一 Pod のサイドカーとして同居**させられる（別ホストにする場合は ARO 上の別 Deployment / Service や Azure Functions も選べる）。
- **ARO はサイドカーに対応する**ので、PDP を同一 Pod のサイドカーに置けば localhost 呼び出しでレイテンシを最小化でき、100〜500ms 目標を守りやすい（別ホストにする場合は 1 ホップを短 TTL キャッシュで吸収。`authorization-models-and-standards-2026.md` のデプロイ注意も参照）。
- **AuthZEN 準拠の PEP として書いておく**と、最初は埋め込み・後から外部、の乗り換えが PEP 側コード変更なしでできる。

### (2) action / resource の語彙を両経路で統一する

FastAPI が `action="read"`、抽出処理が `action="export"` のようにバラバラの語彙を使うと、ポリシー側で両方を書く羽目になり共用の意味が薄れる。**「read という 1 つの action に、出力先（JSON/ファイル）は context として渡す」**ように語彙を設計段階で揃える。② の認可クライアントに寄せると統一しやすい。

### (3) 属性取得（Open-GIM 参照）も共用する

PDP に渡す subject の属性（役職・所属・原価センタ・居住地）は Open-GIM から取得。経路ごとに別実装すると属性の解釈がズレる。`attributes.py` に共通化し、短 TTL の LRU キャッシュもここに集約（`platform-architecture-decision.md` R6 と整合）。

### (4) 列マスク（CLS）も同じ Decision から取る

行フィルタだけでなく、列マスク指示も PDP の判定出力に載せられる（AuthZEN も応答にマスク指示を載せる設計を許容）。FastAPI はレスポンス整形時にマスク適用、抽出処理はファイル書き出し時にマスク適用と、**適用箇所は違うが指示の出所は同じ Decision**にする。

### (5) 監査も共通化

「実際に適用した WHERE と判定理由」を残すのは既存方針の肝。`audit.py` を両経路で共有し、JSON 経路でもファイル経路でも同じスキーマでログする（`row-level-filtering-layering.md`、`authorization-boundaries-and-interface.md` と整合）。

---

## 6. 成立の前提と境界（重要）

- 本メモ（①）は **「利用者が素の BigQuery 直接接続をしない」**ことが前提。利用者は「API 接続」と「抽出依頼」はできるが、自分の手で BigQuery に直接つなぐことはしない。抽出も DB 直結ではなく、PDP を通るサーバ側処理への依頼として実現する。
- 利用者アプリが **自前で BigQuery に DB 接続したい**要件がある場合、その経路は PDP を通らないため①では効かせられない。その場合は別方式が必要：
  - **(A) 接続方式が DB 風なだけ**なら、生テーブルではなく **ポリシー適用済みの認可ビュー／公開データセットにだけ DB 接続させる**ことで①に吸収しうる（ただし per-user 動的出し分けは不可、スライス単位）。
  - **(B) 任意 SQL をエンドユーザー ID で投げたい**なら、ポリシーを正本に **RLS/CLS を生成・同期する方式（②）**。RLS は ID 起点で効くため、エンドユーザー ID 直結なら成立。整合は突合テストで担保。
  - **(C) 任意 SQL を単一 SA で投げたい**は、**ユーザー単位の行制御が原理的に成立しない**（全行が SA に見える）。認可を効かせるなら要件を (A)/(B) に戻す必要がある。
- 切り分けの決め手は **「任意 SQL か／繋ぐのはエンドユーザー ID か SA か」**の 2 点。

---

## 7. 推奨の方向性（叩き）

1. **現行は PyCasbin（埋め込み）で確定**。経路間共用が将来ハード要件に格上げされた場合に限り、**外部型（Cerbos / OPA）を第一候補**として再検討し、ポリシーを唯一の正本にする。ARO ではサイドカー同居が可能なので、まず**同一 Pod のサイドカー**を検討し、必要に応じて別ホスト＋短 TTL キャッシュにフォールバックする。
2. **②認可クライアントと③AST→BigQuery SQL 翻訳層を `authz_core` として共有ライブラリ化**。FastAPI と抽出処理の双方がこれに依存する形にする。これが「真の共用」の本体。
3. **action/resource 語彙の統一・属性取得の共通化・列マスクと監査の共通化**まで揃える。
4. 利用者アプリの「BigQuery 直接接続」要件の**正体（任意 SQL か／ID は誰か）を先に確定**し、①単独か②併用かを決める。

---

## 8. 未決事項 / 次アクション

本メモに関わる未決事項（将来 PDP を外部化する場合の Cerbos/OPA 比較、翻訳層実装、外部 PDP ホスティング、action/resource 語彙、直接接続要件の確定、PoC）は、対応パターン・推奨・実施内容とともに **[`open-issues-and-options.md`](open-issues-and-options.md) に集約**した（論点3・5・6・7、§6 PoC）。本メモが第一候補とする外部 PDP（Cerbos/OPA）は、論点5 の **OPA 見直し閾値に到達した場合の将来選択肢**である（現行は PyCasbin で確定）。

---

## 参考（出典）

- 同じ YAML ポリシーで API アクセスとデータフィルタの両方を駆動（単一の真実の源）: https://www.cerbos.dev/ecosystem/drizzle
- Cerbos PlanResources（クエリプラン＝WHERE 相当を返し述語を DB に押し下げ）: https://www.cerbos.dev/blog/what-is-data-authorization
- Cerbos PDP（ステートレス・サブミリ秒・エアギャップ対応・サーバーレス/サイドカー自己ホスト）: https://www.cerbos.dev/product-cerbos-pdp
- API ゲートウェイ／データプラットフォーム／AI エージェント横断の認可実施: https://www.cerbos.dev/
- Cerbos vs OPA（OPA は汎用ポリシーエンジン、サイドカー配置で低レイテンシ）: https://www.cerbos.dev/blog/cerbos-vs-opa
- 権限認識データフィルタリング（クエリプランで条件を取得し取得時に適用）: https://www.cerbos.dev/features-benefits-and-use-cases/permission-aware-data-filtering
- BigQuery 列レベルアクセス制御・データポリシー API（CLS／マスキング管理）: https://cloud.google.com/bigquery/docs/reference/libraries-overview
