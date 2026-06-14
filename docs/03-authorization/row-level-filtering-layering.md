# Row-Level Filtering Layering

> ステータス: 調査メモ（アーキテクチャ判断材料）
> スコープ: **読み取り（GET）系の行方向フィルタリング**の層配置
> 前提: 認証=Entra ID、認可属性=Open-GIM（社内ユーザーリポジトリ）、データ=BigQuery、API=FastAPI（ARO 上のコンテナ）、**API→BigQuery は単一サービスアカウント（SA）でアクセス**
> Related: `authorization-strategy.md`, `open-issues-and-options.md`（未決論点の正本）, `../90-research/authorization-models-and-standards-2026.md`
> 情報時点: 2026年5月

---

## 略語・用語

略語・用語は [`glossary.md`](glossary.md) を参照（認可ドキュメント共通の正本）。

---

## 0. 結論（先に）

- **認可の「判断」は API／ポリシー層（PDP）が持つ。BigQuery ネイティブの RLS（行アクセスポリシー）は per-user の主役にしない。**
  理由：BigQuery の RLS は「クエリを実行したプリンシパルの ID」で判定するが、今回は API が単一 SA で BigQuery を叩くため、全端末ユーザーが BigQuery からは同一の SA に見え、ユーザー単位の行制御が成立しない。
- **絞り込みの「執行」は、生成した WHERE を BigQuery に押し下げる（pre-filter／パラメータ化）。** アプリで全件取得してメモリ上で捨てる（post-filter）のは厳禁（コスト・性能の両面で悪手）。
- **BigQuery 層の役割は土台整備**：認可ビュー／認可データセットで共通データモデルを「公開単位」として出し、SA を最小権限化（生テーブルは隠す）。フィルタ列をパーティション／クラスタに合わせて、注入した WHERE を安く効かせる。列は CLS（ポリシータグ）で多層防御。
- **一言で**：**判断は API（ABAC）、執行は BigQuery（押し下げ WHERE）、土台はテーブル設計（認可ビュー＋パーティション）。**

---

## 1. 前提となるリクエストの形

```
[人間ユーザー / SA]
      │  Entra ID JWT
      ▼
[APIM] ── 認証検証・スロットル・粗いキャッシュ
      ▼
[FastAPI / ARO]
   - Entra ID で本人特定
   - Open-GIM から認可属性取得（役職・所属・居住地・原価センタ…）
   - ABAC ポリシー評価 → フィルタ条件を生成
   - 単一 SA で BigQuery にパラメータ化クエリ
      ▼
[BigQuery] ── 認可ビュー / テーブル（パーティション・クラスタ）
```

ポイントは「端末ユーザーの文脈（誰が・どの部署・どの役職）は **API 層にしか存在せず、BigQuery には単一 SA としてしか伝わらない**」こと。ここが層配置の判断を決定づける。

---

## 2. 各層に「行絞り込み」を持たせた場合の評価

### 2.1 APIM 層 → 不向き

APIM はゲートウェイであり、得意なのは認証検証・スロットリング・ルーティング・短期キャッシュといった粗い制御。行の中身（部署・起票者など）で絞るのは責務外で、ここに持たせると APIM がデータの意味を知る必要が出て責務が崩れる。per-user の WHERE が付くレスポンスは APIM 組込みキャッシュのヒット率も低い。→ **行絞り込みはしない**。

### 2.2 BigQuery 層（ネイティブ機能）

| 機能 | 何ができるか | 今回の構成での適否 |
|---|---|---|
| **RLS（行アクセスポリシー）** | テーブルに WHERE 相当のフィルタを付与。**クエリ実行者の ID（grantee-list／SESSION_USER）で判定**し、該当しない行は静かに除外 | **per-user の主役にできない**。単一 SA 経由だと端末ユーザーを区別不能。ポリシーは静的でリクエスト毎パラメータを受けない。MV を RLS テーブルに作っても MV の性能利点が消える。RLS をパーティション／クラスタ列にかけるとプルーニングが効かずコスト悪化。Storage Read API 利用時は TRUE フィルタ（実質全件可視）が必要 |
| **認可ビュー／認可データセット** | 別データセットのビューに原本テーブルの読取り権を与え、利用者にはビューだけ見せる。SA は「ビュー側データセット」にだけ権限を持てばよく、生テーブルへの直接権限が不要 | **共通データモデルの公開単位・SA 最小権限化に最適**。ただし行の出し分けは静的（スライスごとに別ビューが要る）→ per-user 動的フィルタの主役にはできない |
| **列レベル（CLS：ポリシータグ／動的データマスク）** | 列単位のアクセス／マスクを保存層で制御 | **多層防御として有効**。ただし IAM プリンシパル基準のため SA 経由では端末ユーザー単位にならない。API 層のマスクと併用する位置づけ |

要するに BigQuery ネイティブの行・列制御はいずれも「**叩いた IAM プリンシパル**」を起点にする。今回は API がプロキシして単一 SA で叩くので、端末ユーザーの粒度には届かない。

### 2.3 FastAPI / ARO 層 → ここが per-user 行制御の主役

端末ユーザーの属性（部署・役職・居住地・起票者）を持てるのはこの層だけなので、必然的にここが**判断点（PDP/PEP）**になる。流れは、Entra ID で本人特定 → Open-GIM で属性取得 → ABAC ポリシー評価 → **フィルタ条件を生成** → BigQuery にパラメータ化 WHERE として渡す。

生成は自由 SQL ではなく、前回調査の方式（**Cerbos PlanResources／OPA Compile の部分評価 → AST → BigQuery SQL への翻訳**）で行う。これにより「オーナーが書くのは宣言的ポリシー、SQL を組み立てるのは翻訳層」という分離ができ、インジェクション・検証・レビューの問題（論点③）を構造的に解消できる。

---

## 3. 「押し下げ」が肝：pre-filter vs post-filter

- **post-filter（全件取得してアプリで捨てる）は厳禁**。BigQuery は**スキャンしたバイト数で課金**（返した行数ではない）。全件読んでから捨てると、無駄なスキャン課金とアプリのメモリ／I/O 浪費が同時に発生する。
- **pre-filter（WHERE を BigQuery に押し込む）が正解**。さらにフィルタ列を**パーティション／クラスタに合わせる**と、パーティションプルーニングで実スキャン量とコストが大きく下がる（フィルタが効くと「読まないパーティション」は課金対象から外れる）。
- **注意**：パーティション列を関数で包む（例：`EXTRACT(MONTH FROM ts)=2`）とプルーニングが効かない。生成する WHERE は `partition_col = @val` のような**直接比較**の形にする。実装後は dry-run でスキャンバイトを確認し、プルーニングが効いているか検証する。

---

## 4. 推奨レイヤリング

| 層 | 行絞り込みにおける責務 |
|---|---|
| **APIM** | 認証（JWT）検証・スロットル・粗いキャッシュ。**行絞り込みはしない** |
| **FastAPI / ARO** | **判断（ABAC）** と **WHERE 生成（パラメータ化）**。列マスクの適用。監査（適用した条件の記録）。属性は Open-GIM 参照＋短 TTL キャッシュ |
| **BigQuery** | **執行**（押し下げ WHERE を実行）。加えて **認可ビュー／認可データセット**で共通モデル公開・SA 最小権限・生テーブル隠蔽、**パーティション／クラスタ**で安く効かせる、**CLS** で保存層の多層防御 |

```
判断（誰が何を見てよいか）  →  FastAPI / ARO（ABAC, PDP）
執行（実際に行を絞る）      →  BigQuery（pre-filter された WHERE を実行）
土台（安く・安全に）        →  認可ビュー + パーティション/クラスタ + CLS
```

---

## 5. パラメータ化と安全性（論点③への直結）

- WHERE を**文字列連結で組まない**。ユーザー由来の値（部署コード等）は BigQuery の**クエリパラメータ**（`@param`、配列なら `UNNEST(@allowed_depts)`）で渡し、インジェクションを防ぐ。
- 生成するのは「属性 → 条件」だけ。ユーザーが任意 SQL を書ける口は作らない。ポリシー（宣言的）→ 翻訳層（AST→パラメータ化 SQL）の一方向に固定する。

---

## 6. キャッシュとの関係（`../02-architecture/platform-architecture-decision.md` と整合）

- per-user の WHERE が付くレスポンスは共有キャッシュのヒット率が低い（R2）。**キャッシュは共通参照系（マスタ・スキーマ・公開メタデータ）に限定**し、per-user 結果は短 TTL／プロセス内に留める。
- 属性（Open-GIM）はプロセス内 LRU で短 TTL キャッシュし、毎リクエストの問い合わせを避ける（R6）。

---

## 7. 監査

RLS のような「黙って絞る」挙動を API 層で自前実装するため、**実際に適用した WHERE／条件**と判定理由を監査ログに必ず残す（論点⑧）。BigQuery を直接叩く経路が併存する場合は、BigQuery 側の Data Access ログや `INFORMATION_SCHEMA.ROW_ACCESS_POLICIES` も併用する。

---

## 8. 例外：いつ BigQuery RLS を使うか

API を介さず **BigQuery を直接叩く利用者**（BI／アナリストがコンソールや Looker で素のクエリを投げる）が併存する場合、その経路には RLS が有効。彼らは自分の Google／Workforce Identity Federation の ID で叩くため、RLS が ID を起点に効く。つまり「**API 経路＝API 層 ABAC、直接経路＝BigQuery RLS**」の二経路併用はあり得る。ただし行ルールの二重管理コストに注意（`../02-architecture/platform-architecture-decision.md` の「二重管理回避」と整合させ、原則は API 層を正本にする）。

---

## 参考（出典）

- BigQuery 行レベルセキュリティ概要（行アクセスポリシー＝ grantee-list と filter_expression、WHERE 相当）: https://docs.cloud.google.com/bigquery/docs/row-level-security-intro
- 行アクセスポリシーの管理（IAM 前提、JSON 列不可など）: https://docs.cloud.google.com/bigquery/docs/managing-row-level-security
- RLS は実行ユーザーの ID で判定／一致しなければ 0 行（SESSION_USER 例）: https://oneuptime.com/blog/post/2026-02-17-how-to-set-up-row-level-security-in-bigquery-using-row-access-policies/view
- RLS ベストプラクティス（パーティション／クラスタ列に RLS をかけるとプルーニングが効かない）: https://docs.cloud.google.com/bigquery/docs/best-practices-row-level-security
- RLS と他機能（MV では RLS により MV の性能利点が消える、Storage Read API は TRUE フィルタ必要）: https://docs.cloud.google.com/bigquery/docs/using-row-level-security-with-features
- 認可ビュー（別データセットのビューに原本読取り権、利用者にはビューのみ、最高性能）: https://docs.cloud.google.com/bigquery/docs/authorized-views
- 認可データセット（SA は A データセットの権限だけで B を読める＝最小権限）: https://pawait.africa/google-cloud-update-august-25-changes-to-dataset-level-access-controls-on-bigquery/
- 認可ビュー単体では行の動的出し分けにならない（スライスごとに別ビュー）: https://www.devoteam.com/expert-view/3-options-to-protect-bigquery-data-with-row-level-security/
- 列レベルアクセス制御（ポリシータグ、認可ビューでも基底列の CLS が効く）: https://docs.cloud.google.com/bigquery/docs/column-level-security-intro
- BigQuery はスキャンバイト課金（行数でない）／不要列・不要行も読めば課金: https://dev.to/amaranarun/how-to-optimize-bigquery-costs-real-techniques-that-work-1c0h
- パーティションプルーニング（パーティション列への直接フィルタでスキャン縮小）: https://cloud.google.com/bigquery/docs/querying-partitioned-tables
- プルーニングが効かない例（パーティション列を関数で包むと全走査）: https://oneuptime.com/blog/post/2026-02-17-how-to-optimize-bigquery-query-performance-by-eliminating-full-table-scans-with-partitioning/view
- フィルタ押し下げ（pre-filter）の考え方（取得してから捨てない）: https://www.cerbos.dev/blog/pre-vs-post-filtering-for-authorization
