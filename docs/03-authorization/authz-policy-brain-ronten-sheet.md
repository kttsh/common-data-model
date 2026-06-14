# 共通データモデルに対する認可ポリシー統一方式の論点整理
― BigQuery RLS / CLS・BigQuery Sharing・API PDP の役割分担 ―

> 副題：*“ポリシーの頭脳を1つにする” とは何を意味するのか*
>
> 対象：共通データモデルに対する**参照アクセス制御**。API 側 PDP と BigQuery 側 RLS/CLS（データ層強制）、および BigQuery sharing（旧 Analytics Hub）をどう束ねるか。
> スコープ前提：**参照系のみ／検証用の簡単な公開API／部門費モデルでのパイロット／x・y 部門への展開フレーム確立**。
> 最終更新：2026-06-14（レビュー結果 `authz-policy-brain-review.md` 反映版）

---

## 本版の変更点（レビュー反映サマリ）

`authz-policy-brain-review.md` の指摘を次のとおり反映した。

- 構成を**結論先出し**に変更（§0 エグゼクティブサマリを新設し、推奨構成・採用しない案・未確定事項・成功条件を冒頭に置いた）。
- 正式タイトルを社内資料向けに硬くし、「頭脳を1つに」は副題に降格。
- カジュアル表現を置換（「いちばん効く事実」→「本検討の分岐点」、「罠」→「制約／注意点」、「ギミック」→「実現パターン」、「ドリフトを殺す」→「ドリフトを検知・防止する」、「推し」→「第一候補／推奨」、「productズ化」→「製品化」）。
- `filteredDataViewer` を「注意」から**設計原則**へ昇格。SA の `TRUE` filter を**例外権限**として明示。
- subquery RLS の制約を推奨案の直後に集約し、「成立する」→「**制約付きで成立する／パイロットで検証する**」と表現を慎重化。
- OpenGIM/オンプレ属性同期は**日次で十分**という前提を明記。第一候補を日次バッチ（オンプレ抽出→GCS→BQ定期ロード or BigLake 外部表）とし、Datastream/CDC は将来選択肢に位置付けた（原シートも実は同順だが、サマリで明示）。
- 「世の中のプラクティス」「派閥マップ」「詳細比較」「原コメント」「出典」は**付録**へ集約し、本文は意思決定に必要な要点に圧縮。
- 追加論点：**論点7（日次同期 SLO）／論点10（BI 接続方式）／論点11（説明可能性）**。

> ⚠️ レビューが扱っていない**プロジェクト内の矛盾**を 2 点、本文の該当箇所（論点1・論点3・論点9）に「要確認」として明示した。要旨はチャットでも別途報告する。

---

## 0. エグゼクティブサマリ

本検討の目的は、共通データモデルに対する参照権限を、利用者・部門・API ごとの個別実装ではなく、**モデル単位のポリシー**として一元管理できるかを検証することである。

**推奨構成（部門費モデルのパイロット）**

1. **ワンマートを複製しない。** BigQuery 上の単一の共有テーブルを保ち、整ったビューを意味層として公開する。テナント（部門）ごとのテーブル複製は恒久的な保守コストになる［7］。
2. **参照は 2 チャネル並列**で、同じワンマートを別ドアから見る。
   - **BQ ネイティブ直参照チャネル**（BI／アナリストが自分の identity で接続）：**Authorized View ＋ BigQuery RLS/CLS** で行・列を強制。BigQuery Sharing の linked dataset で配信。
   - **API 参照チャネル**（API が単一 SA で BigQuery を叩く）：**API 層 ABAC（PDP）が判断**し、生成した行フィルタ（WHERE）を BigQuery クエリに**押し下げて執行**。列はレスポンス整形時にマスク。
3. **認可判断の属性**は、Entra ID 認証後の利用者 ID と OpenGIM 由来の組織・役職属性を用いる。ただし BigQuery はクエリ時にオンプレ OpenGIM を直接参照できないため、OpenGIM 属性は **BigQuery 側の mapping 表または Cloud Identity グループとして実体化**する必要がある。
4. **OpenGIM/オンプレ属性の同期は、部門費パイロットでは日次で十分。** 第一候補は「オンプレ抽出 → GCS → BigQuery 定期ロード」または「BigLake 外部表」。Datastream（CDC）は分単位の鮮度が要件化した場合の**将来選択肢**とする。
5. **回帰防止**：approved fixtures（被験者×文脈 → 見えるべき行/列）を CI で API と BigQuery の両方に当て、差分が出たら落とす。

**研究性／新規性**は、RBAC/ABAC/ReBAC や OPA/RLS といった個別技術そのものではなく、**モデル単位に一度定義した認可ポリシーを、人事マスタ由来の属性で評価し、BigQuery と API の両チャネルへ一貫適用する運用フレームワーク**を確立し、職改・異動時の権限メンテをゼロにできることを部門費モデルで実証する点にある（→ 論点9 末尾）。

**未確定・要確認事項（先出し）**

- ★**認可エンジンの方針**：本シートは「単一ポリシー（decision-shape 統一）」を狙い OPA 系を推奨するが、**プロジェクトの決定の正本では PyCasbin（埋め込み）が採用済み**であり、整合確認が必要（→ 論点1・論点9 の「要確認」）。
- ★**強制点の“正本”の呼び方**：データ層を権威とするか、API 層を正本とするか。2 チャネルを分けると整合する（→ 論点3 の「要確認」）。
- BI／分析クライアントが**利用者本人**で接続するか**共有 SA**で接続するか（→ 論点10）。
- 日次同期の **SLO と失敗時の扱い**（前日スナップショット継続 or 参照停止 or セキュア側に倒す）（→ 論点7）。
- mapping 表の**プライバシー**（subquery 型 RLS では参照表にも `getData` が要り、他人の本部コードが見え得る）（→ 論点6 注意1）。

---

## 1. 今回のスコープ

- **参照系のみ**（更新系は対象外。権限は参照権限のみ。C5 の確認に対する明言）。
- **部門費モデルでのパイロット**（VC本 × ST部門費の行サブセット提供を新方式で検証）。
- **BQ ネイティブ参照と API 参照の 2 チャネル**を対象。API が対象にするのは **Publisher 側のテーブル/ビュー**、Subscriber Project は BQ ネイティブ直参照チャネル専用で API は触らない（C3 の答え）。
- x・y 部門へ展開するための**フレーム確立**が最終ゴール（〈モデル＋属性 → 行/列スライス〉の再利用テンプレート）。

---

## 2. 推奨アーキテクチャ

| 構成要素 | 役割 | 強制点 |
|---|---|---|
| ワンマート | 単一の共有テーブル（複製しない）［7］ | ― |
| Authorized View / BigQuery Sharing | 公開面・配信（linked dataset は読み取り専用ポインタ、複製ではない）［1,6］ | ― |
| BigQuery RLS | 行制御の**本体**。基底テーブルに `filter_expression`＋`SESSION_USER()`＋lookup subquery［2,3］ | データ層（直参照チャネル） |
| BigQuery CLS（policy tag / data masking） | 列制御の**本体**。行レベルと完全併用可［5］ | データ層 |
| OpenGIM 属性 mapping 表（`ref` データセット） | 認可属性の実体化（email → 本部コード等）。RLS subquery の参照先 | データ層の入力 |
| API PDP ＋ 押し下げ WHERE | API チャネルの**判断**と執行（単一 SA 前提） | API 層 |
| approved fixtures（CI） | 2 チャネルの判定差分を検知し回帰を防止 | テスト |

- **行制御の本体は RLS、列制御の本体は policy tag / CLS / data masking** であり、**Authorized View 自体は行・列制御の本体ではなく公開面**である（誤解防止のための明確化）。
- **直参照チャネル**は RLS/CLS が利用者の identity を起点に効く。**API チャネル**は単一 SA で叩くため、判断は API 層 ABAC が持ち、フィルタを WHERE として押し下げて執行する（→ 論点3）。

---

## 3. Google 公式仕様に基づく成立条件・制約

> ここは「方針」ではなく**仕様上の制約**を集約する章。推奨構成が「制約付きで成立する」ことを示す。

### 3.1 linked dataset（複製しない）
BigQuery Sharing のリスティングを購読すると、Subscriber project に **linked dataset** が作られる。linked dataset は共有データセットへの**読み取り専用ポインタ／参照であり、コピーではない**［1］。linked dataset は `bigquery.user` だけで参照でき、**購読自体がアクセス許可として働く**ため、Publisher 側 IAM を増やさず展開できる［6］。

### 3.2 Authorized View（公開面）
Authorized View は、ソーステーブルへの直接アクセスを与えずにビュー経由でデータを共有する仕組み。ビューは**別データセット**に置き、ソース側で**明示的に authorize** し、両者は**同一ロケーション**が必須［1,9］。**Authorized View 単体は行レベル制御ではない**（全行見える）。`SESSION_USER()` や RLS と組み合わせて初めて行レベルになる［10］。

### 3.3 RLS / `SESSION_USER()` / subquery（制約をまとめて）
付与先（ユーザー/グループ/ドメイン/SA）＋ `filter_expression`（WHERE 相当）を基底テーブルに定義し、クエリ実行時に評価する［2］。`SESSION_USER()`＋lookup 表 subquery で「ユーザーごとに複数ポリシーを作らない」設計が可能：`region IN (SELECT region FROM lookup WHERE email = SESSION_USER())`［3］。

**subquery を含む RLS の制約（推奨案の直後に明示）**：

- subquery を含む行アクセスポリシーは**追加課金**を生み得る［2］。
- RLS のフィルタは**パーティション/クラスタの pruning に参加しない**ため、読み取り/課金行が増え得る［2］。
- subquery RLS は **BigQuery Storage Read API（単純述語のみ）と非互換**［2,5］。
- テーブルの **preview / browse は RLS と非互換**［2］。
- subquery RLS が**参照できるのは BigQuery テーブル／BigLake 外部表／BigLake managed テーブルに限られる**［2］。
- subquery 結果には **100MB 制限**がある［2］。
- **data masking は非サブクエリ型 RLS とのみ併用可**［2］。

> **設計上の判断**：mapping 表型 RLS は、OpenGIM 属性を BigQuery 側で評価するための有力案である。ただし上記制約のため、部門費パイロットでは**性能・課金・BI 接続方式・data masking 併用可否を必ず検証**する。

### 3.4 CLS / data masking
ポリシータグ＋動的データマスキングで列単位を制御。**行レベルと列レベルは完全併用可**［5］。Authorized View 経由でも基底列の CLS は効く。

### 3.5 filteredDataViewer（**設計原則**）
行アクセスポリシーを作成すると、BigQuery が grantee に `bigquery.filteredDataViewer` を**自動付与**する。この role を IAM で直接付与してはならない［4］。

> **原則**：`roles/bigquery.filteredDataViewer` は **row access policy 経由でのみ付与し、IAM で直接付与しない**。テーブル/データセット/プロジェクトに広く IAM 付与すると全行が見える。

### 3.6 TRUE filter（**例外権限**）
フルアクセスが必要な service account job などは `TRUE` filter を使える［2］。これは全行参照に近い強い権限であるため、通常権限ではなく**例外権限**として扱う。

> **原則**：ETL/運用 SA に `TRUE` filter を付与する場合は、通常利用者とは別の例外権限として扱い、**対象 SA・用途・実行ジョブ・監査ログ確認方法を台帳化**する。

### 3.7 整合（ドリフト源）
テーブルをコピーすると RLS/CLS もコピーされるが、**コピー先は元と非同期（独立）**になりドリフト源になる［5］。**複製しない**方針はこの観点でも妥当。

---

## 4. 主要論点

各論点 = **問い／回答パターン・選択肢／世の中の派閥・落としどころ**。

### 論点0：BigQuery Sharing をどう使うか（または使わないか）
**問い**：DB 参照チャネルを BQ Sharing（Publisher/Subscriber・linked dataset）で配るか、Authorized View 跨プロジェクト＋IAM だけで十分か。Subscriber Project の作成と SA 発行/管理をどう設計するか。RLS は linked dataset 経由で購読者に効くのか。

**回答パターン**
- **(a) BQ Sharing を採用**：Subscriber が自己購読 → Publisher 側 IAM を増やさず展開できる［6］。提供先が増えるほど運用が軽い。Exchange/Listing/linked dataset／Subscriber Project／SA のガバナンスが増えるのがコスト。
- **(b) Authorized View 跨プロジェクト＋IAM のみ**：提供先が少数固定ならこれで足りる。Sharing の機構を持たない分シンプル。
- **(c) ワンマート＋View（複製しない）**：(a)(b) いずれでも前提とする作り［7］。
- **(d) SA 方針**：人間は属性（グループ/identity）→ RLS で捌き、SA はシステム間連携限定・最小権限・鍵レス(WIF)・IaC 管理。フルアクセスが要る SA ジョブは `TRUE` filter（例外権限）。

**落としどころ（公正化）**：提供先が増える／Subscriber 側で購読管理したい／組織横断で配信したい場合は **BQ Sharing**。提供先が少数固定で運用が単純なら **Authorized View＋IAM のみ**も比較対象とする。実務は**グループ粒度**で組むのが定石［8］。**検証必須**：RLS を基底テーブルに置いたとき、linked dataset 経由でも購読者の identity／所属グループに対して正しく評価されるか（VC本 × ST部門費で確認）。

### 論点1：何を「1つ」にするのか（spec / runtime / decision-shape）
**問い**：統一の対象は正本仕様か、実行時 PDP か、判定の出力（行フィルタ）か。
**回答パターン**：①spec のみ統一（実装 2 つ＋テスト）／②runtime 統一（1 PDP）／③decision-shape 統一（脳が SQL を吐く）。
**本検討の分岐点**：②③は「脳が行フィルタ（SQL）を返せるか」に依存する。PyCasbin の出力は真偽（または許可オブジェクトの列挙）で **SQL の WHERE 句を生成しない**［11］。一方 OPA は部分評価で **SQL/UCAST を生成できる**［12,13］。つまり②③を狙うなら Casbin 単独は苦しく、OPA（または相当の仕組み）が要る。

> ⚠️ **要確認（プロジェクトの決定との整合）**：本プロジェクトの**認可エンジンの正本**は **PyCasbin（埋め込み）で採用済み**である（`04-research/abac-authz-library-comparison.md`＝採用決定の正本、`02-architecture/platform-architecture-decision.md` D2、`03-authorization/shared-pdp-across-api-and-bigquery.md`）。外部 PDP（OPA/Cerbos）は**「見直し閾値」付きの将来選択肢**と位置付けられている（閾値＝ABAC の `eval()` matcher が概ね 10〜15 ルールを超える／行レベル SQL・UCAST フィルタや列マスクの**ネイティブ生成**が必要になった場合）。
> したがって本シートの「OPA で decision-shape 統一」は、**現行採用（PyCasbin）の置き換えを断定するものではなく**、部門費パイロットで「単一脳（1 つのポリシーから両強制を導く）」の成立性を検証し、**D2 見直しの判断材料を得る研究提案**として読む。採用を OPA に倒すかは別途 D2 の再決定が必要。

### 論点2：何を判定するのか――粒度と関心の分離
**問い**：1 つの脳に統合すべきか、粗い判定（エンドポイント/アクション/テナント）と細かい判定（行・列・セル）で役割分担すべきか。
**回答パターン**：(a) 完全統合 / (b) **2 つの脳・別の仕事**（API＝粗い、データ層＝細かい）。
**落としどころ**：Casbin はアプリの粗い認可に好適。行・列はデータ層に寄せる「役割分担」も正解。C5 の「3 つのロール」は **役職／所属／個人の 3 属性軸**と読み替え、権限は**参照のみ**と明言。

### 論点3：権威ある強制点はどこか（信頼境界・多重防御）
**問い**：データ層強制とアプリ層強制のどちらを権威とし、どう多重防御するか。直 SQL／Sharing 購読者の素通りをどう扱うか。
**回答パターン**：(a) データ層を権威（どのクライアントも一律） / (b) アプリ層を権威（API 集約、BQ は API 用 SA に施錠） / (c) 両層で多重防御。
**落としどころ**：直 BQ 参照を許す限り、**データ層に最低 1 枚（RLS/CLS）は外せない**。`filteredDataViewer` の広域 IAM 付与は厳禁［4］（→ §3.5）。

> ⚠️ **要確認（強制点の“正本”）**：本シートはデータ層（RLS/CLS）を権威ある強制点として前面に置く。一方、`03-authorization/row-level-filtering-layering.md` は「**API → BigQuery は単一 SA**」を前提に、**per-user の行制御は BigQuery ネイティブ RLS の主役にできない**（全端末ユーザーが同一 SA に見える）／**判断は API 層 ABAC、執行は押し下げ WHERE、BQ RLS は直接接続者向けの例外**としている。
> 両者は **2 チャネルを分ける**と整合する：**直参照・Sharing 購読者（自分の identity で叩く）には RLS/CLS が効き、API チャネル（単一 SA）は API 層 ABAC＋押し下げ WHERE**。どちらを「正本」と呼ぶか（データ層 or API 層）を**明示して用語を統一**すること。

### 論点4：ポリシー統一の実現パターン――6 パターン（中核）
**問い**：どの方式で頭脳を共通化するか。

| # | パターン | 一言 | 頭脳の所在 | 両チャネルを 1 脳で覆えるか | Casbin | OPA |
|---|---|---|---|---|---|---|
| 1 | チャネル統合 | 全アクセスを API に集約、直 BQ/Sharing 廃止 | API の PDP | ◎（直 BQ を消すので自明） | ○ | ○ |
| 2 | コンパイル/トランスパイル | 正本 1 つから API 強制＋**BQ 行アクセスポリシー DDL** を生成 | ビルド時生成器＋正本 | ◎ | △ | ○ |
| 3 | 中央 PDP＋部分評価 → SQL pushdown | PDP が WHERE を返し API の BQ クエリに付加 | OPA（必須級） | △（**API チャネル内のみ**） | ✕ | ◎ |
| 4 | policy-as-data（共有マッピング表） | 組織階層/可視性を BQ 参照表に置き、RLS は `SESSION_USER()`＋subquery、API も同表参照 | 共有データ＋小ルール | ○（ロジックは SQL 止まり） | ○ | ○ |
| 5 | 二重実装＋適合テスト | 別々に作り、ゴールデン期待値を CI で両者に当て差分検出 | 正本 spec＋テスト | ○（runtime 非統一） | ○ | ○ |
| 6 | 標準プロトコル(AuthZEN) | PEP↔PDP を標準化し判定をサービス化 | 外部 PDP | （2/3 と併用前提） | ○ | ◎ |

**エビデンス**：#3 は OPA compile が SQL/UCAST を返す［12,13］、Permit/Styra が製品化［14,15］。ただし**直 BQ は覆えない**。両チャネル統一は #2（正本 → BQ DDL も生成）か、直 BQ を消す #1。#4 は BQ subquery RLS が lookup 表参照を可能にする［3］が課金/pruning 制約あり［2］。「買う」版の #2＝Immuta（1:1 secure view を生成）［25］。

### 論点5：表現力の最小公倍数――各層で何が書けるか
**問い**：単一化できる範囲は「両層で表現できる条件」の交差。role_memo の条件は各層で書けるか。

| 条件 | BQ RLS / CLS | PyCasbin | OPA(Rego) |
|---|---|---|---|
| 自部門のみ（所属・原価センタ） | ◎ filter＋lookup subquery | △ 真偽は出るが WHERE は自前 | ◎ 部分評価で WHERE 生成 |
| 主席以上＝全社（役職しきい値） | ○ 役職をグループ/表で表現 | ◎ matcher 比較 | ◎ |
| **予算列＝全員／実績列＝管理職のみ**（列×役職） | ◎ CLS（タグ＋マスキング） | △ 列概念は自前 | ○ EOPA は列マスキング compile |
| 日本非居住者は不可（属性） | ○ 居住区分を表に持てば可 | ◎ | ◎ |
| 上位組織まで閲覧可（関係） | △ 階層表で近似 | △ | ○ / ReBAC なら◎ |
| 申請不要の全社デフォルト | ○ ドメイン付与＋RLS | ◎ デフォルトポリシー | ◎ |

**落としどころ**：**行＝RLS、列＝CLS、しきい値/属性＝ルール**の分担が素直。C6 の「予算 vs 実績」は**行（自部門）×列（予算は全員／実績は管理職のみ）の積**で表現する（p6 の主役例）。Casbin は行・列とも自前で WHERE／列選択を組む必要があるため、これがもう一つの真実源になりやすい。

### 論点6：属性の供給（PIP）と「BQ への属性到達」――隠れた律速
**問い**：OpenGIM 属性をどの強制点にどう届けるか。**オンプレにしか無い属性**（例「この SA は xx 本部＝コード値」）を BQ 側でどう使えるようにするか。
**事実**：アプリには request-time で届く（context に積んで PDP へ）。OPA は外部データを自分で取りに行けない［16］。**BQ がクエリ時に見える「誰」は `SESSION_USER()`（メール）とその IAM グループ所属のみ**で、オンプレ照会もリクエスト属性の受け渡しも不可。

**BQ 側での実体化（2 択）**
- **(a) メールキーの mapping 表**：`ref.principal_attributes(email, honbu_code, …)` を置き、RLS subquery で `honbu_code IN (SELECT honbu_code FROM ref.principal_attributes WHERE email = SESSION_USER())`。SA が叩けば `SESSION_USER()` は SA メールを返すので `(sa-xx@…iam.gserviceaccount.com,'1234')` で表現できる。
- **(b) Cloud Identity グループ所属**：本部ごとにグループを作り `GRANT TO ('group:…')`。Entra 認証なら Entra → Google Cloud Identity のグループ・プロビジョニングが追加で要るため、(a) の方が素直なことが多い。

**federation では回避できない**：`EXTERNAL_QUERY`（federated query）が繋げるのは AlloyDB／Spanner／Cloud SQL のみで、結果はネイティブ表ではなく**一時テーブル**として返る［31］。オンプレ DB には直接届かず、subquery 型 RLS が参照できるのは BigQuery／BigLake テーブルに限られる［2］ため、federation の一時結果は RLS 内で使えない。**よって同期がほぼ必須**。

**同期手段（オンプレ → BQ／マスタ系＝小・低頻度を前提）**

| 方式 | 中身 | 向く状況 | 鮮度 |
|---|---|---|---|
| **バッチ（第一候補）** | オンプレ抽出 → GCS → **BQ 定期ロード**（or GCS 上ファイルを **BigLake 外部表**＝RLS subquery 可）。Composer/Workflows/Scheduler＋Cloud Run で実行 | 組織マスタの小・低頻度 | 日次〜時次 |
| マネージド CDC | **Datastream → BQ**（Oracle/MySQL/PostgreSQL/SQL Server、オンプレ可、insert/update/delete＋バックフィル）。要 Cloud VPN/Interconnect［29,30,32］ | 分単位の鮮度が要る | 秒〜分 |
| Dataflow / 3rd party | Dataflow JDBC → BQ、Fivetran/Airbyte(OSS)/Striim/Debezium | 既存基盤・非対応ソース | 構成次第 |
| イベント駆動 | オンプレが変更イベント → Pub/Sub → BQ subscription | 即時反映 | 秒 |
| グループ経路((b)用) | オンプレ組織 → Cloud Identity（AD/LDAP は GCDS、IdP は SCIM） | グループ型 RLS | 方式次第 |

**落としどころ／第一候補**：組織・識別子マッピングは小さく低頻度なので、**「オンプレ抽出 → GCS → BQ 定期ロード（or BigLake 外部表）」で日次が第一候補**。`ref` データセットを統制対象として RLS subquery の参照先にする。分単位が要件化したら Datastream（将来選択肢）。

**注意（2 点）**
1. **mapping 表は認可データ＝センシティブ**。subquery 型 RLS では付与先が対象表だけでなく**参照表にも `bigquery.tables.getData` を要する**［3］ので、**消費者が mapping を直接読める**（他人の本部コードが見える）。開示可否を判断し、嫌なら本部ごと `GRANT TO group` の**非サブクエリ形**／authorized view で包む。マスキングは**非サブクエリ型 RLS とのみ併用可**［2］。`filteredDataViewer` を広域 IAM 付与しない［3,4］。
2. **鮮度＝不整合窓（論点7）**。バッチは最大 1 サイクル遅延、CDC は秒〜分。RLS subquery は課金・pruning 非対応なので **mapping 表は小さく保つ**［2］。

### 論点7：OpenGIM 属性の日次同期 SLO（整合性・鮮度）★レビュー追加
**問い**：部門費パイロットでは同期は**日次で十分**。この前提のもとで、職改・異動の反映窓をどこまで許すか。同期失敗時にどう倒すか。
**経路**：BQ＝group 反映／mapping 表リフレッシュ／DDL 再デプロイ。
**落としどころ**：
- **第一候補は日次バッチ同期**（オンプレ抽出 → GCS → BQ 定期ロード or BigLake 外部表）。
- 日次同期で**職改・異動の反映遅延が業務上許容される**ことを確認する。
- **同期失敗時の扱いを決める**：前日スナップショットを継続利用するのか、対象 API／BQ 参照を止めるのか、セキュリティ側に倒すのか。
- 「API は（属性キャッシュ TTL に応じ）相対的に新鮮、BQ は表更新まで遅延」等の不整合窓を **SLO として明示**し許容範囲を決める。

### 論点8：オーサリング/ガバナンス(PAP)・デプロイ結合・ドリフト検知
**問い**：正本ポリシーの置き場・版管理・監査・テストをどう設計するか（C8）。
**回答パターン**：正本を Git/カタログに（CODEOWNERS＝データオーナー＋セキュリティ）、変更は git commit で監査［18］。**モデル単位の成果物（policy-as-product）**としてカタログ登録。
**ドリフト検知**：`harness-engineering-guide` の「計算的センサー＋approved fixtures」を流用＝**〈被験者＋文脈 → 見えるべき行/列〉のゴールデン期待値を API と BQ の両方に CI で当て、差分が出たら落とす**（ドリフトを検知・防止する）。どのパターンを採っても回帰防止に併用。

### 論点9：買うか作るか＆エンジン選定（メタ）
**問い**：統制プレーン製品を買うか、自前で作るか。作るならどのエンジンか。
**回答パターン**：
- **買う**：Immuta/Privacera。一発で「1 脳 → 多 DB」を得るが年 \$10万〜\$50万+級で研究用途には過剰［26］。Immuta は AND/OR 混在不可の指摘あり（出典は競合 Privacera＝バイアスあり、**要自己検証**）［27］。
- **作る**：研究スコープ（検証用公開 API）はこちら。エンジン選定は下記「要確認」に従う。

> ⚠️ **要確認（論点1 と同件）**：エンジンの**現行採用は PyCasbin（埋め込み）**で確定（D2／`abac-authz-library-comparison.md`）。本シートが推す OPA は「単一ポリシーから両強制（行 SQL／列マスク）をネイティブ生成」したい場合の将来選択肢であり、**部門費パイロットはその成立性検証と D2 見直し材料の取得**と位置付ける。
> - **PyCasbin 路線（現行）**：エンドポイント＋アクション ABAC は PyCasbin、`row_filter`／列マスクは `Decision` を返す**自前層＋ AST → BigQuery SQL 翻訳層**で補完。直参照チャネルは BQ RLS/CLS が担う。
> - **OPA 路線（将来選択肢）**：Rego を正本に、部分評価で API 用 WHERE を生成（#3）し、可能なら #2 で BQ 行アクセスポリシー DDL も生成。Cedar は静的解析が強み［22］だが Python バインディング・SQL 生成は未成熟。
> - **移行の閾値**：ABAC の `eval()` matcher が概ね 10〜15 ルール超／テスト・保守が困難化／行レベル SQL・UCAST・列マスクのネイティブ生成が必要化（OPA≥v1.9.0 Compile API）。

### 論点10：BI / 分析クライアントの接続方式 ★レビュー追加
`SESSION_USER()` ベースの RLS は、BigQuery にクエリを発行する**主体**に依存する。BI ツールが**利用者本人**で接続するのか、**共有 SA**で接続するのかを確認する。共有 SA の場合、`SESSION_USER()` は SA メールになるため、利用者単位の ABAC は別方式（API 経由＋押し下げ WHERE 等）が必要になる。

### 論点11：認可判定の説明可能性 ★レビュー追加
運用に入ると「なぜこの行が見えないのか」「なぜこの列がマスクされるのか」という問い合わせが必ず発生する。OpenGIM 属性 → mapping 表 → RLS policy → CLS policy tag のどこで許可/不許可が決まったかを**追跡できる**ようにする。パイロットでは、代表ユーザーについて「見える理由／見えない理由」を説明できることを検証する。

---

## 5. パイロット検証項目（部門費 / VC本 × ST部門費）

- [ ] ワンマート上の Authorized View＋RLS で「ST部門費の行サブセット」を VC本に提供できるか（複製マートなし）。
- [ ] BQ Sharing の linked dataset 経由で、**RLS が購読者の identity／所属グループに正しく効くか**（まずグループ粒度）。
- [ ] `SESSION_USER()`＋OpenGIM 由来 mapping 表の subquery RLS が機能するか／課金・pruning 影響の実測（§3.3 の制約を確認）。
- [ ] 列の出し分け（予算＝全員／実績＝管理職のみ）が CLS（ポリシータグ＋マスキング）で表現できるか。data masking と subquery 型 RLS の併用制約も確認。
- [ ] 同一ポリシー（採用エンジンに応じて PyCasbin or Rego）から、API の BQ クエリ用 WHERE が想定どおり生成されるか。★エンジン方針（論点1/9）の決定後に確定。
- [ ] approved fixtures（被験者×文脈 → 見えるべき行/列）を API と BQ の両方に当て、差分ゼロを確認。
- [ ] BQ Sharing を「使う/使わない（Authorized View＋IAM のみ）」の運用コスト比較。
- [ ] Subscriber Project と SA の発行・最小権限・鍵レス(WIF)・IaC 化の手順確立。
- [ ] OpenGIM/オンプレ属性を **BigQuery 側の mapping 表に日次同期**できることを確認（第一候補＝オンプレ抽出 → GCS → BQ 定期ロード or BigLake 外部表）。
- [ ] **日次同期で、職改・異動の反映遅延が業務上許容**されることを確認。
- [ ] **同期失敗時**に、前日スナップショット継続／参照停止／セキュリティ側に倒す、のいずれにするか決定（論点7）。
- [ ] **Datastream は分単位の鮮度が要件化した場合の将来選択肢**として比較対象に留める。
- [ ] mapping 表のプライバシー：subquery RLS で**参照表に `getData` が要る**ため消費者が mapping を読めてしまう点の可否確認（嫌なら非サブクエリ形／authorized view で包む）。
- [ ] federation（`EXTERNAL_QUERY`）はオンプレ非対応・RLS subquery 非対応である前提を確認し、同期方式と**鮮度 SLO**（職改の反映窓）を決定。
- [ ] BI／分析クライアントが利用者本人で接続するか共有 SA かを確認（論点10）。
- [ ] 代表ユーザーで「見える理由／見えない理由」を説明できるか（論点11）。

---

## 付録A：レビュアー原コメント → 論点・方針 対応表

レビュアーの指摘（原文は付録 A-2）を論点と現時点の方針に紐づけたもの。

| # | コメント要旨 | 関連論点 | 現時点の方針（たたき台） |
|---|---|---|---|
| C1 | DB参照は BQ Sharing で、複製マートを作らず**ワンマートのまま View**で対応を検証 | 論点0, 3, 4#4 | 妥当。Authorized View＋RLS＋CLS で「複製しない」を実現。VC本×ST部門費の行サブセットで検証 |
| C2 | Subscriber Project の作り方・**SA の発行/管理**が今後の検討事項 | 論点0, 6, 7 | 人間は属性（グループ/identity）→ RLS、SA はシステム間連携限定・最小権限・鍵レス(WIF)・IaC 管理 |
| C3 | API提供は Subscriber/Publisher の両方が対象か、**Publisher のテーブルに適用**か | 論点0, 3 | API が対象にするのは **Publisher 側のテーブル/ビュー**。Subscriber Project は BQ ネイティブ直参照チャネル用で API は触らない |
| C4 | RBAC は職改で棚卸し必要 → EntraID 認証後 **OpenGIM 属性で判定（ABAC）**、ReBAC も？ **新規性をアピール** | 論点5, 6, 9 | ABAC を主役に。ReBAC は関係が主役の要件が出たら追加。新規性は §0/論点9 末尾に集約 |
| C5 | p7：その**3 つのロール**だけで普遍管理できる確認？ 権限は**参照のみ**？ | 論点2, 5 | 「3 つのロール」ではなく「**役職／所属／個人の 3 属性軸**」。代表モデルで網羅検証。権限は参照のみ（明言） |
| C6 | p6 を丁寧に＋p5 と統合。**予算は全員・実績は一部**のパターン | 論点5 | 行＝RLS（自部門）×列＝CLS（予算は全員／実績は管理職のみ）の積で表現 |
| C7 | p9 の**認可テーブル**だと今のシステムと変わらない。**ルールエンジン**で管理すべき | 論点1, 4, 9 | 静的テーブルではなく**ポリシー（ルール）エンジン**。人 → 可否の事前付与表は持たない |
| C8 | そのルール(SQL可)を**カタログに登録**して再利用 | 論点8 | PAP＝カタログ登録された再利用可能ポリシー（policy-as-product）。Git/カタログに正本、CODEOWNERS 管理 |
| C9 | 研究は**検証用の簡単な公開API**で十分。部門費で試し、**x/y 部門への当てはめフレーム**確立 | 論点9, §0 | 「買う」ではなく「作る」。部門費で〈モデル＋属性 → 行/列スライス〉の再利用テンプレートを確立 |

### 付録A-2：レビュアー原コメント（原文）

```
・DB参照に対しては、Big Query sharingを使って、BQ内にマート(参照用テーブル)を作るのではなく、ワンマートのままViewで対応するのを検証するで良いと思います。
　Subscriber Projectをどう作成して、SAを発行・管理していくとかが、今後の検討事項かと。
　丁度、VC本に対して、ST部門費管理のサブセット(行レベルで抽出)をして提供予定なので、それを新方式で対応したらどうなるか確認出来るかと。
・で、API提供は、Subscriber ProjectとPublisher Projectの両方を対象にしますか？むしろ、Publisher Project内のテーブルに適用でしょうか？
・例えば、RBACでは所属ロールを作成して、そこに所属する人をそのロールにあらかじめ個人を登録して管理している。これだと、人事異動・職制改正のたびにロールの棚卸が必要。
　なので、EntraIDで認証後、OpenGIMから、その人の属性を取得して、その属性で判断するABAC(属性＋ルール？)と組みあわせる事で、柔軟な対応が可能か検証するって事ですかね？
　後、ReBAC（Relationship-Based Access Control）というのもあるらしい(Copilot先生)ので、これらを組みあわせる？
　新規性(研究性)がどの辺にあるのかアピールして欲しい。
・p7は、その3つのロールだけで普遍的に管理出来る事を確認するって事？ここで言う権限は、参照権限のみ？
・p6を、もう少し丁寧に書いて欲しい。P5の説明と組みあわせたら良いかなと。
　(例えば、予算は所属全員に見せるけど、実績は一部にしか見せないみたいなパターン無いかな～？)
・p9の認可テーブルを持つなら、あまり、今の色々なシステムでやっているのと変わらないような？
　ここを、ルールエンジンみたいなもので、静的テーブルではなくルールで管理するとした方が良いような。
　(これも、やっているかもしれませんが)
・このルール(SQLでも良いけど)みたいのをカタログに登録しておいて、それを利用するような形になればと。
・こっちの研究では、検証用の簡単な公開APIが出来れば良いので、例えば、部門費管理を利用して試していけば良いかなと。
　(x部門やy部門に提供する場合に、こうなるというような考え方(フレームワーク)が確立出来ればと)
```

---

## 付録B：世の中のプラクティス（2026）詳細

### B.1 BigQuery Sharing / Authorized View / RLS / CLS
- **BigQuery Sharing（旧 Analytics Hub）**：Publisher が Exchange 上に Listing を公開 → Subscriber が購読すると、自プロジェクトに**読み取り専用の linked dataset** が作られる［1］。linked dataset は `bigquery.user` だけで参照でき、購読自体がアクセス許可として働く［6］。
- **ワンマート＋View（複製しない）は推奨形**：テナント（部門）ごとのテーブル複製は「永久に払う保守税」になる［7］。
- **Authorized View の要点と注意点**：別データセット・明示 authorize・同一ロケーションが必須。ビュー作成だけでは不十分［9］。単体は行レベル制御ではない（全行見える）［10］。
- **RLS（行アクセスポリシー）**：付与先＋ `filter_expression` を基底テーブルに定義し、クエリ実行時に評価［2］。`SESSION_USER()`＋lookup 表 subquery で設計可能［3］。制約は §3.3 に集約。
- **filteredDataViewer**：自動付与。広域 IAM 付与は厳禁（§3.5）。RLS は**組織内**用途に推奨（課金情報がサイドチャネルになり得る）［4］。
- **CLS（列レベル）**：ポリシータグ＋動的データマスキング。行レベルと完全併用可［5］。
- **グループ粒度の運用**：行レベルをグループ単位で管理し、購読者をグループに追加して許可行だけ見せるのが定石。プロジェクト構成は Ingestion → Curated → DataExchange(=Publisher) の機能別分割が標準［8］。

### B.2 RBAC / ABAC / ReBAC（2026 のコンセンサス）
- 3 モデルは排他ではなく、多くの本番系は **RBAC（粗い）＋ReBAC（リソース単位）のハイブリッド**に落ち着き、ABAC はエッジケースを担う［21］。
- **ReBAC は RBAC の上位集合**で、属性を関係として表せば ABAC のシナリオも covered。OpenFGA は Conditions / Contextual Tuples で拡張［19,20］。
- 時刻・地理・分類レベルのような条件は **OPA/Rego の ABAC が idiomatic**。グリーンフィールドは RBAC から始め、ロール爆発が制約になったら FGA を入れる［21］。

### B.3 外部化認可 / Policy-as-Code（PEP・PDP・PAP）
- 認可ロジックを外部化すれば、ランタイムで認可でき、変更が **git commit として監査**できる［17,18］。
- アーキは **PAP / PDP / PEP** の 3 点。PDP は Cedar や OPA で自前構築可［16］。**Cedar**（AWS 発）は静的解析に強い［22］。**OPA** は Rego・REST・外部データ取り込み可だが、**自分で外部データを取りに行けない**ので属性は push/キャッシュ or 判定時取得で input に渡す［16］。**Casbin** は軽量・アプリ埋め込み向き［23］。

### B.4 OPA 部分評価 → SQL（単一脳×データフィルタの鍵）
- OPA の **Compile API** が部分評価で残余条件を返し、**SQL WHERE 句 / UCAST** として出力（Accept ヘッダで選択）［12,13］。
- 周辺実装：Styra Enterprise OPA は**列マスキングの compile** も提供［14］、Permit.io は SQL 自動翻訳を製品化［15］。
- **重要な限界**：これは「API の SA が投げるクエリに WHERE を足す」もので、**BQ に行アクセスポリシーを設置はしない**。直 SQL／Sharing 購読者は覆えない。

### B.5 統制プレーン製品（買う選択肢）
- **Immuta**：BigQuery では**ポリシープッシュ型**で、各テーブルと 1:1 の secure view を作る。行・列・セルレベル＋ABAC＋目的ベース［25,26］。価格は中規模で年 \$10万〜\$20万、エンタープライズで \$50万+［26］。
- **Privacera**：多プラットフォーム統制（オープン標準/Ranger 系）［27］。Immuta の AND/OR 制約の指摘は出典が競合のため**要自己検証**［27］。

### B.6 比較対象：Databricks Unity Catalog ABAC
- governed タグ＋UDF＋マッピング表で ABAC 化、session-user の identity で評価［28］。1 テーブルにつきユーザーごと有効な行フィルタは 1 つ（衝突はエラー）。SecureView バリアで述語プッシュダウンが阻まれ、定数 true でも全表スキャンになり得る → **BQ の「非 pruning／課金増」と同型の制約**として認識する［28］。

### B.7 標準化：AuthZEN（将来の布石）
- **AuthZEN Authorization API 1.0**（OpenID Foundation、2026 年 3 月公開）が PEP↔PDP の通信を JSON REST で標準化。ポリシー言語は規定せず OPA/Cedar 等がそのまま使え、Search API も持つ［24］。

### B.8 オンプレ → BQ の属性同期（PIP の実体化）
論点6 に集約。要旨：BQ がクエリ時に見える「誰」は `SESSION_USER()`＋IAM グループのみ。federation では回避できず、**同期がほぼ必須**。マスタ系＝小・低頻度のため**日次バッチが第一候補**、分単位が要れば Datastream（将来選択肢）。

---

## 付録C：世の中の派閥マップ（要約）

| 派閥 | 代表 | 思想 | この件への含意 |
|---|---|---|---|
| 外部化PDP / Policy-as-Code | OPA, Cedar, Topaz, Axiomatics, AuthZEN | 認可をコード化し 1 PDP に集約・脱結合 | パターン1/3/6。**OPA の部分評価**が単一脳×SQL の鍵 |
| データ中心セキュリティ | BQ RLS/CLS, Snowflake RAP, Databricks UC ABAC | 強制をデータ層へ。どのクライアントも一律 | パターン0/4。プッシュダウン阻害＝全表スキャンの制約［28］ |
| 統制プレーン製品 | Immuta, Privacera | 1 ポリシー → 各 DB へ自動反映（secure view 等） | パターン2 の「買う」版［25,26］ |
| 関係/Zanzibar | OpenFGA, SpiceDB | タプル＆グラフで per-object 権限 | 行・列フィルタ生成は不得手。本件は当面 ABAC 優位 |
| 軽量ライブラリ埋め込み | **PyCasbin**, casl, oso | アプリ内 in-process の手軽さ | 粗い/アプリ認可に最適。**行・列の単一化は不得手**。本プロジェクトの**現行採用** |
| 仕様＋適合テスト/Harness | approved fixtures, fitness functions | runtime 非統一・spec＋テストで縛る | パターン5。どの構成でも回帰防止に併用 |

---

## 付録D：出典（Google 公式リファレンス等）

1. Google Cloud — Authorized views / BigQuery sharing. https://docs.cloud.google.com/bigquery/docs/authorized-views
2. Google Cloud — Introduction to row-level security（課金・Storage API 非互換・TRUE フィルタ）. https://docs.cloud.google.com/bigquery/docs/row-level-security-intro
3. Google Cloud — Use row-level security（SESSION_USER／lookup subquery）. https://docs.cloud.google.com/bigquery/docs/managing-row-level-security
4. Google Cloud — Best practices for row-level security（filteredDataViewer・組織内用途）. https://docs.cloud.google.com/bigquery/docs/best-practices-row-level-security
5. Google Cloud — Using row-level security with other features（copy 非同期・CLS 併用・data masking）. https://docs.cloud.google.com/bigquery/docs/using-row-level-security-with-features
6. classmethod / DevelopersIO — BigQuery Sharing permissions（linked dataset は bigquery.user のみ）. https://dev.classmethod.jp/en/articles/bigquery-sharing-permission-points/
7. Medium (N. Rajput) — BigQuery Row-Level Policies + Views（複製しない）. https://medium.com/@hadiyolworld007/bigquery-row-level-policies-views-tenant-isolated-analytics-without-duplicate-tables-f588ab41c3da
8. Medium / Google Cloud Community (S. Karadag) — Manage Access to Analytics Hub + Authorized Views（グループ粒度・プロジェクト構成）. https://medium.com/google-cloud/how-to-manage-access-bigquery-analytics-hub-and-authorized-views-e38b47e6a0b9
9. OneUptime — Create BigQuery Views and Authorized Views（別データセット・authorize・同一ロケーション）. https://oneuptime.com/blog/post/2026-02-17-how-to-create-bigquery-views-and-authorized-views-for-secure-data-sharing/view
10. Devoteam — 3 ways to protect BigQuery data with RLS（アプリ vs ウェアハウス・非 pruning）. https://www.devoteam.com/expert-view/3-options-to-protect-bigquery-data-with-row-level-security/
11. Casbin — Data Permissions（implicit assignments / BatchEnforce は真偽配列）. https://www.casbin.org/docs/data-permissions/
12. Open Policy Agent — REST API Reference（Compile API・SQL/UCAST・Accept ヘッダ）. https://www.openpolicyagent.org/docs/rest-api
13. Open Policy Agent — Data filtering / Partial evaluation（unknowns → 残余条件 → SQL）. https://www.openpolicyagent.org/docs/filtering/partial-evaluation
14. Styra — Data Filters Compilation API（Enterprise OPA・列マスキング compile）. https://docs.styra.com/enterprise-opa/reference/api-reference/partial-evaluation-api
15. Permit.io — Data Filtering（Source-Level / Partial Evaluation・SQL 翻訳）. https://docs.permit.io/how-to/enforce-permissions/data-filtering/
16. AWS Prescriptive Guidance — PDP via OPA / PAP・PDP・PEP / Cedar・OPA. https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/opa.html
17. Axiomatics — Externalized Authorization (EAM). https://axiomatics.com/resources/reference-library/externalized-authorization
18. ssojet — Authorization as a Service and Policy-as-Code（PEP/PDP・git 監査）. https://ssojet.com/ciam-qna/authorization-as-a-service-and-policy-as-code
19. OpenFGA — Authorization Concepts（ReBAC は RBAC の上位集合・Conditions/Contextual Tuples）. https://openfga.dev/docs/authorization-concepts
20. Auth0 — Understanding ReBAC and ABAC Through OpenFGA and Cedar. https://auth0.com/blog/rebac-abac-openfga-cedar/
21. CIAM Compass — RBAC vs ABAC vs ReBAC（2026 ハイブリッド）. https://guptadeepak.com/ciam-compass/guides/rbac-vs-abac-vs-rebac/
22. StrongDM — Cedar Policy Language (CPL): 2026 Complete Guide. https://www.strongdm.com/cedar-policy-language
23. Permit.io — Top Open-Source Authorization Tools for Enterprises in 2026. https://www.permit.io/blog/top-open-source-authorization-tools-for-enterprises-in-2026
24. DEV (kanywst) — AuthZEN Authorization API 1.0 Deep Dive（OpenID, 2026-03）. https://dev.to/kanywst/authzen-authorization-api-10-deep-dive-the-standard-api-that-separates-authorization-decisions-1m2a
25. Immuta — Google BigQuery Integration Documentation 2026.1（policy push・1:1 secure view）. https://documentation.immuta.com/2026.1/configuration/integrations/google-bigquery
26. Modern DataTools — Immuta Review (2026)（単一統制プレーン・価格）. https://www.modern-datatools.com/tools/immuta
27. Privacera — Privacera vs Immuta（多プラットフォーム・AND/OR 制約の指摘＝要検証）. https://privacera.com/privacera-vs-immuta/
28. Databricks — Unity Catalog ABAC: requirements / performance / policies. https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/requirements
29. Google Cloud — Datastream for BigQuery. https://cloud.google.com/datastream-for-bigquery
30. Google Cloud — Datastream overview（対応ソース・プライベート接続）. https://docs.cloud.google.com/datastream/docs/overview
31. Google Cloud — Introduction to federated queries（`EXTERNAL_QUERY` は AlloyDB/Spanner/Cloud SQL のみ・結果は一時テーブル）. https://docs.cloud.google.com/bigquery/docs/federated-queries-intro
32. OneUptime — Replicate Oracle changes to BigQuery using Datastream（オンプレは VPN/Interconnect 必須）. https://oneuptime.com/blog/post/2026-02-17-replicate-oracle-database-changes-bigquery-datastream/view

### プロジェクト内参照（整合確認用）
- `04-research/abac-authz-library-comparison.md`（**認可ライブラリ比較・PyCasbin 採用決定の正本**）
- `02-architecture/platform-architecture-decision.md`（§8.2 D2＝PyCasbin 確定、改訂履歴 1.2）
- `03-authorization/shared-pdp-across-api-and-bigquery.md`（PDP 共用、PyCasbin 確定済み・外部 PDP は将来）
- `03-authorization/row-level-filtering-layering.md`（単一 SA 前提・判断は API 層・BQ RLS は直接接続者向け例外）
- `03-authorization/pycasbin/`（基本仕様・BQ への row_scope 展開実装）
