# 権限ポリシーの「頭脳を1つに」――論点シート（コメント × プラクティス × 論点）

> 対象：共通データモデルに対する参照アクセス制御。PyCasbin（API側 PDP）と BigQuery 側 RLS/CLS（データ層強制）、および BigQuery sharing（旧 Analytics Hub）をどう束ねるか。
> スコープ前提：**参照系のみ／検証用の簡単な公開API／部門費モデルでのパイロット／x・y 部門への展開フレーム確立**。
> 最終更新：2026-06-14

---

## 0. 全体の補助線（このシートの読み方）

レビュアーのコメントは、突き詰めると次の3点に収斂する。各論点はこの3軸のどこかに対応する。

1. **どこで強制するか** … BigQuery（データ層）か API（アプリ層）か、両方か。
2. **何で判定するか** … 静的な認可テーブルか、ルールエンジンか。
3. **何を根拠にするか** … 事前付与（職改で棚卸し発生）か、人事マスタ（OpenGIM）由来の属性か。

そして「**ポリシーの頭脳を1つに**」には3つの意味があり、どれを狙うかでエンジン選定まで決まる（→ 論点1）。

- **仕様(spec)を1つに**：人が読む正本を1つにし、実装は2つでも可（テストで縛る）。
- **実行時(runtime)を1つに**：1つの PDP が両強制点に効く。
- **出力(decision shape)を1つに**：頭脳が「許可/不許可」だけでなく「行フィルタ(SQL)」を吐き、両者で使う。

**いちばん効く事実**：PyCasbin の出力は真偽（または許可オブジェクトの列挙）で、**SQL の WHERE 句を生成しない**［11］。一方 OPA は部分評価で **SQL/UCAST を生成できる**［12,13］。つまり上記②③を狙うなら Casbin 単独は苦しく、OPA（または相当の仕組み）が要る。repo-strategy が PDP 候補を「OPA-WASM / Cedar / casbin 等」と併記しているため、ここはまだ動かせる決定である。

---

## 1. コメント → 論点・方針 対応表

レビュアーの指摘（原文は付録A）を論点と現時点の方針に紐づけたもの。

| # | コメント要旨 | 関連論点 | 現時点の方針（たたき台） |
|---|---|---|---|
| C1 | DB参照は BQ sharing で、複製マートを作らず**ワンマートのまま View**で対応を検証 | 論点0, 3, 4#4 | 妥当。Authorized View＋RLS＋CLS の三点セットで「複製しない」を実現。VC本×ST部門費の行サブセットで検証 |
| C2 | Subscriber Project の作り方・**SA の発行/管理**が今後の検討事項 | 論点0, 6, 7 | 人間は属性（グループ/identity）→RLS、SA はシステム間連携に限定・最小権限・鍵レス(WIF)・IaC 管理 |
| C3 | API提供は Subscriber/Publisher の両方が対象か、**Publisher のテーブルに適用**か | 論点0, 3 | API が対象にするのは **Publisher 側のテーブル/ビュー**。Subscriber Project は「BQネイティブ直参照」チャネル用で API は触らない |
| C4 | RBAC は職改で棚卸し必要 → EntraID 認証後 **OpenGIM 属性で判定（ABAC）**、ReBAC も組み合わせ？ **新規性をアピール**して | 論点5, 6, 9 | ABAC を主役に。ReBAC は関係が主役の要件が出たら追加。新規性は「3つの組合せ」ではなく後述（§5）に置く |
| C5 | p7：その**3つのロール**だけで普遍管理できる確認？ 権限は**参照のみ**？ | 論点2, 5 | 「3つのロール」ではなく「**役職／所属／個人の3属性軸**」。普遍ではなく代表モデルで網羅検証。権限は参照のみ（明言） |
| C6 | p6 を丁寧に＋p5 と統合。**予算は全員・実績は一部**のパターン | 論点5 | 行=RLS（自部門）×列=CLS（予算は全員/実績は管理職のみ）の積で表現。p6 の主役例にする |
| C7 | p9 の**認可テーブル**だと今のシステムと変わらない。**ルールエンジン**で管理すべき | 論点1, 4, 9 | repo-strategy は既にルールエンジン(OPA/Cedar/casbin)。p9 の「テーブル」表現を「ポリシー(ルール)エンジン」に直す。人→可否の事前付与表は持たない |
| C8 | そのルール(SQL可)を**カタログに登録**して再利用 | 論点8 | PAP＝カタログ登録された再利用可能ポリシー（policy-as-product）。Git/カタログに正本、CODEOWNERS 管理 |
| C9 | 研究は**検証用の簡単な公開API**で十分。部門費で試し、**x/y部門への当てはめフレーム**確立 | 論点9, §5 | 「買う(Immuta等)」ではなく「作る」。部門費で〈モデル＋属性→行/列スライス〉の再利用テンプレートを確立 |

---

## 2. 世の中のプラクティス（2026）要約

### 2.1 BigQuery sharing / Authorized View / RLS / CLS

- **BigQuery sharing（旧 Analytics Hub）**：Publisher が Exchange 上に Listing を公開 → Subscriber が購読すると、自プロジェクトに **読み取り専用の linked dataset** が作られる［1］。**linked dataset は `bigquery.user` だけで参照でき、Data Viewer は不要。購読自体がアクセス許可として働く**ため、Publisher 側の IAM を増やさず Subscriber が自分で購読できるのが利点［6］。
- **ワンマート＋View（複製しない）は推奨形**：単一の共有テーブルを保ち、行アクセスポリシーで部門分離し、整ったビューを意味層として公開する。テナント（部門）ごとのテーブル複製は「永久に払う保守税」になる［7］。
- **Authorized View の要点と罠**：ビューはソーステーブルと**別データセット**に置き、ソース側で**明示的に authorize** し、両者は**同一ロケーション**が必須。ビュー作成だけでは不十分［9］。Authorized View 単体は行レベル制御ではなく（全行見える）、`SESSION_USER()` や RLS と組み合わせて初めて行レベルになる［10］。
- **RLS（行アクセスポリシー）**：付与先（ユーザー/グループ/ドメイン/SA）＋ `filter_expression`（WHERE 相当）をベーステーブルに定義し、クエリ実行時に評価［2］。`SESSION_USER()` ＋ lookup 表 subquery で「ユーザーごとに複数ポリシーを作らない」設計が可能：`region IN (SELECT region FROM lookup WHERE email = SESSION_USER())`［3］。
- **RLS の制約**：subquery ポリシーは**追加課金**を生み、**パーティション/クラスタのプルーニングに参加しない**ため読み取り/課金行が増えうる。subquery RLS は **BigQuery Storage Read API（単純述語のみ）と非互換**。テーブルのプレビュー/ブラウズも非互換。SA 等フルアクセスが要るジョブは `TRUE` フィルタで対応［2,5,10］。
- **filteredDataViewer の扱い**：行アクセスポリシーは付与先に `bigquery.filteredDataViewer` を自動付与する。**この役割をテーブル/データセット/プロジェクトに IAM で広く付与すると全行が見えてしまう**ので、必ずポリシー経由で管理。RLS は**組織内**用途に推奨（課金情報がサイドチャネルになり得る）［4］。
- **CLS（列レベル）**：ポリシータグ＋動的データマスキング。**行レベルと列レベルは完全併用可**［5］。
- **グループ粒度の運用**：Analytics Hub × Authorized View では、行レベルをグループ単位で管理し、購読者をそのグループに追加して許可行だけ見せるのが定石。プロジェクト構成は Ingestion → Curated → DataExchange(=Publisher) の機能別分割が標準［8］。

### 2.2 RBAC / ABAC / ReBAC（2026 のコンセンサス）

- 3モデルは排他ではなく、**多くの本番系は RBAC（粗い）＋ ReBAC（リソース単位）のハイブリッド**に落ち着き、ABAC はエッジケースを担う［21］。
- **ReBAC は RBAC の上位集合**で、属性を関係として表せば ABAC のシナリオも covered。OpenFGA は Conditions / Contextual Tuples で残りの属性駆動ケースを拡張する［19,20］。Zanzibar（Google：Drive/YouTube/Calendar/Cloud）が代表実装。本番級は OpenFGA・SpiceDB 等［21］。
- **「時刻・地理・分類レベル」のような条件は OPA/Rego による ABAC の方が idiomatic**（FGA の関係モデルより）。グリーンフィールドは RBAC から始め、ロール爆発が制約になったら FGA を入れる［21］。

### 2.3 外部化認可 / Policy-as-Code（PEP・PDP・PAP）

- 認可ロジックをアプリから分離（外部化）すれば、業務・規制・組織変更でコードを触らずランタイムで認可でき、**変更が git commit として監査**できる［17,18］。
- アーキは **PAP（ポリシー管理）/ PDP（評価）/ PEP（強制）** の3点。PDP は Cedar や OPA で自前構築可［16］。**Cedar**（AWS発）は RBAC/ABAC/ReBAC/*BAC を対象に静的解析に強い［22］。**OPA** は Rego・REST・外部データ取り込み可だが、**自分で外部データを取りに行けない**ので属性は push/キャッシュ or 判定時取得で input に渡す［16］。**Casbin** は軽量・アプリ埋め込み向き（後述の弱点あり）［23］。

### 2.4 OPA 部分評価 → SQL（単一脳×データフィルタの鍵）

- OPA の **Compile API** が部分評価で「未知の値に依存する残余条件」を返し、**SQL WHERE 句 / UCAST** として出力（Accept ヘッダで選択）［12］。`input.<TABLE>.<COLUMN>` を unknown に指定し、既知部分を評価し切って残余をフィルタにする［13］。
- 周辺実装：Spring Boot で JPA/Mongo へ変換するライブラリ群、Styra Enterprise OPA は**列マスキングの compile** も提供［14］、Permit.io は `filterObjects` で SQL 自動翻訳を productズ化［15］。
- **重要な限界**：これは「**API の SA が投げるクエリに WHERE を足す**」もので、**BQ に行アクセスポリシーを設置はしない**。よって直SQL / sharing 購読者（API を通らない者）は覆えない。

### 2.5 統制プレーン製品（買う選択肢）

- **Immuta**：Snowflake/Databricks/BigQuery/Starburst を**1つの統制プレーン**で横断。BigQuery では**ポリシープッシュ型**で、各テーブルと 1:1 の secure view を作りそこに全ポリシーロジックを入れ、ユーザー権限用の Immuta データセットも作る。行・列・セルレベル＋ABAC＋目的ベース、データ経路に入らず、ロール爆発を抑える［25,26］。価格は中規模で年 \$10万〜\$20万、エンタープライズで \$50万+［26］。
- **Privacera**：同様の多プラットフォーム統制（オープン標準/Ranger 系）、Unity Catalog を補完［27］。Immuta の制約として「AND と OR を1文に混在不可」との指摘あり（出典は競合 Privacera＝バイアスあり、要自己検証）［27］。

### 2.6 比較対象：Databricks Unity Catalog ABAC

- governed タグ＋UDF＋マッピング表で ABAC 化、**session-user の identity で評価**［28］。**1テーブルにつきユーザーごと有効な行フィルタは1つ**（衝突はエラー）。**SecureView バリア**で述語プッシュダウンが阻まれ、フィルタが定数 true でも全表スキャンになり得る（perf）。ビューに直接 ABAC は付けられない（下のテーブルのポリシーは尊重）。ETL/パイプラインの SA は EXCEPT で除外［28］。→ **BQ の「非プルーニング／課金増」と同型の罠**であり、データ層強制の一般的コストとして認識する。

### 2.7 標準化：AuthZEN（将来の布石）

- **AuthZEN Authorization API 1.0**（OpenID Foundation、2026年3月公開）が **PEP↔PDP の通信を JSON REST で標準化**（「誰が・何を・どのリソースに」）。ポリシー言語は規定せず OPA/Cedar 等がそのまま使え、リソース列挙用の Search API も持つ［24］。「認可判定を再利用可能なサービスとして外出し」する思想と相性が良い。

### 2.8 オンプレ→BQ の属性同期（PIP の実体化）

- **BQ がクエリ時に見える「誰」は `SESSION_USER()`（メール）とそのIAMグループ所属のみ**。オンプレ照会やリクエスト属性の受け渡しはできない。よって判定に使う属性（例「このSAは xx本部＝コード値」）は、**BQ側に mapping 表 or Cloud Identity グループ所属として実体化**する必要がある（→ 論点6）。
- **federation では回避できない**：BigQuery の `EXTERNAL_QUERY`（federated query）が繋げるのは **AlloyDB / Spanner / Cloud SQL のみ**で、結果はネイティブ表ではなく**一時テーブル**として返る［31］。オンプレDBには直接届かず、**サブクエリ型RLSが参照できるのは BigQuery／BigLake テーブルに限られる**［2］ため、federationの一時結果はRLS内で使えない。Cloud SQLレプリカ＋federationにしても、レプリカへのオンプレ同期が別途要る＝**同期を消すのではなく場所を移すだけ**。→ **同期がほぼ必須**。
- **同期手段**：①バッチ（オンプレ抽出→GCS→BQ定期ロード、or GCS上ファイルを **BigLake外部表**＝RLSサブクエリ可）／②マネージドCDC（**Datastream**＝Oracle/MySQL/PostgreSQL/SQL Server、ソースはオンプレ可、要 Cloud VPN/Interconnect）［29,30,32］／③Dataflow JDBC・3rd party(Fivetran/Airbyte/Striim/Debezium)／④イベント駆動（Pub/Sub→BQ subscription）。詳細・選定・注意点は **論点6**。

---

## 3. 論点シート（BQ sharing を独立論点として先頭に）

各論点 = **問い／回答パターン・選択肢／世の中の派閥・落としどころ**。

### 論点0：BigQuery sharing をどう使うか（または使わないか）★今回追加
**問い**：DB参照チャネルを BQ sharing（Publisher/Subscriber・linked dataset）で配るか、Authorized View 跨プロジェクト＋IAM だけで十分か。Subscriber Project の作成と SA 発行/管理をどう設計するか。RLS は linked dataset 経由で購読者に効くのか。

**回答パターン**
- **(a) BQ sharing を採用**：Subscriber が自己購読 → Publisher 側 IAM を増やさず展開できる［6］。提供先が増えるほど運用が軽い。Exchange/Listing/linked dataset／Subscriber Project／SA のガバナンスが増えるのがコスト。
- **(b) Authorized View 跨プロジェクト＋IAM のみ**：提供先が少数固定ならこれで足りる。sharing の機構を持たない分シンプル。
- **(c) ワンマート＋View（複製しない）**：(a)(b) いずれでも前提とする作り。複製マートは作らない［7］。
- **(d) SA 方針**：人間は属性（グループ/identity）→RLS で捌き、**SA はシステム間連携限定・最小権限・鍵レス(WIF)・IaC 管理**。フルアクセスが要る SA ジョブは `TRUE` フィルタ免除［2］。

**世の中の落としどころ / 派閥**
- 「データ層に強制を寄せ、どのクライアントも一律」=データ中心セキュリティ派の本流。sharing はその配信手段。
- **検証必須事項**：RLS をベーステーブルに置いたとき、linked dataset 経由でも**購読者の identity / 所属グループに対して正しく評価されるか**。実務は**グループ粒度**で組むのが定石［8］。これを VC本×ST部門費の行サブセットで確認する（C1）。
- **API との関係（C3 の答え）**：API が対象にするのは **Publisher のテーブル/ビュー**。Subscriber Project は「BQネイティブ直参照」チャネル専用で、API はそこを触らない。**2つのチャネル（BQネイティブ／API）は並列**で、同じワンマートを別ドアから見る。

### 論点1：何を「1つ」にするのか（spec / runtime / decision-shape）
**問い**：統一の対象は正本仕様か、実行時 PDP か、判定の出力（行フィルタ）か。
**回答パターン**：①spec のみ統一（実装2つ＋テスト）／②runtime 統一（1 PDP）／③decision-shape 統一（脳が SQL を吐く）。
**落としどころ**：②③は「脳が SQL を返せるか」に依存。**Casbin は不可、OPA は可**［11,12］。まずここを合意してから比較する。

### 論点2：何を判定するのか――粒度と関心の分離
**問い**：1つの脳に統合すべきか、粗い判定（エンドポイント/アクション/テナント）と細かい判定（行・列・セル）で役割分担すべきか。
**回答パターン**：(a) 完全統合 / (b) **2つの脳・別の仕事**（API=粗い、データ層=細かい）。
**落としどころ**：Casbin はアプリの粗い認可に好適。行・列はデータ層に寄せる「役割分担」も正解。C5 の「3つのロール」は**役職／所属／個人の3属性軸**と読み替え、権限は**参照のみ**と明言。

### 論点3：権威ある強制点はどこか（信頼境界・多重防御）
**問い**：データ層強制とアプリ層強制のどちらを権威とし、どう多重防御するか。直SQL/sharing 購読者の素通りをどう扱うか。
**回答パターン**：(a) データ層を権威（どのクライアントも一律） / (b) アプリ層を権威（API 集約、BQ は API用SAに施錠） / (c) 両層で多重防御。
**落としどころ**：直BQ参照を許す限り、**データ層に最低1枚（RLS/CLS）は外せない**。`filteredDataViewer` の広域 IAM 付与は厳禁［4］。

### 論点4：単一化の「ギミック」――6パターン（中核）
**問い**：どの方式で頭脳を共通化するか。

| # | パターン | 一言 | 頭脳の所在 | 両チャネルを1脳で覆えるか | Casbin | OPA |
|---|---|---|---|---|---|---|
| 1 | チャネル統合 | 全アクセスを API に集約、直BQ/sharing 廃止 | API の PDP | ◎（直BQを消すので自明） | ○ | ○ |
| 2 | コンパイル/トランスパイル | 正本1つから API強制＋**BQ行アクセスポリシーDDL**を生成 | ビルド時生成器＋正本 | ◎ | △ | ○ |
| 3 | 中央PDP＋部分評価→SQL pushdown | PDP が WHERE を返し API の BQ クエリに付加 | OPA（必須級） | △（**APIチャネル内のみ**） | ✕ | ◎ |
| 4 | policy-as-data（共有マッピング表） | 組織階層/可視性を BQ 参照表に置き、RLS は `SESSION_USER()`＋subquery、API も同表参照 | 共有データ＋小ルール | ○（ロジックは SQL 止まり） | ○ | ○ |
| 5 | 二重実装＋適合テスト | 別々に作り、ゴールデン期待値を CI で両者に当て差分検出 | 正本 spec＋テスト | ○（runtime非統一） | ○ | ○ |
| 6 | 標準プロトコル(AuthZEN) | PEP↔PDP を標準化し判定をサービス化 | 外部PDP | （2/3 と併用前提） | ○ | ◎ |

**エビデンス**：#3 は OPA compile が SQL/UCAST を返す［12,13］、Permit/Styra が productズ化［14,15］。ただし**直BQは覆えない**。両チャネル統一は #2（正本→BQ DDL も生成）か、直BQを消す #1。#4 は BQ subquery RLS が lookup 表参照を可能にする［3］が課金/プルーニング制約あり［2］。**「買う」版の #2 = Immuta**（1:1 secure view を生成）［25］。

### 論点5：表現力の最小公倍数――各層で何が書けるか
**問い**：単一化できる範囲は「両層で表現できる条件」の交差。role_memo の条件は各層で書けるか。

| 条件 | BQ RLS / CLS | PyCasbin | OPA(Rego) |
|---|---|---|---|
| 自部門のみ（所属・原価センタ） | ◎ filter＋lookup subquery | △ 真偽は出るが WHERE は自前 | ◎ 部分評価で WHERE 生成 |
| 主席以上＝全社（役職しきい値） | ○ 役職をグループ/表で表現 | ◎ matcher 比較 | ◎ |
| **予算列=全員／実績列=管理職のみ**（列×役職） | ◎ CLS（タグ＋マスキング） | △ 列概念は自前 | ○ EOPA は列マスキング compile |
| 日本非居住者は不可（属性） | ○ 居住区分を表に持てば可 | ◎ | ◎ |
| 上位組織まで閲覧可（関係） | △ 階層表で近似 | △ | ○ / ReBAC なら◎ |
| 申請不要の全社デフォルト | ○ ドメイン付与＋RLS | ◎ デフォルトポリシー | ◎ |

**落としどころ**：**行=RLS、列=CLS、しきい値/属性=ルール**の分担が素直。C6 の「予算 vs 実績」は行（自部門）×列（予算は全員/実績は管理職のみ）の積。Casbin は行・列とも自前 WHERE/列選択になり、実質もう一つの真実源になりやすい。

### 論点6：属性の供給（PIP）と「BQ への属性到達」――隠れた律速
**問い**：OpenGIM 属性をどの強制点にどう届けるか。**オンプレにしか無い属性**（例「このSAは xx本部＝コード値」）を、BQ側でどう使えるようにするか。
**事実**：アプリには request-time で届く（context に積んで PDP へ）。OPA は外部データを自分で取りに行けない［16］。**BQ がクエリ時に見える「誰」は `SESSION_USER()`（メール）とそのIAMグループ所属のみ**で、オンプレ照会もリクエスト属性の受け渡しも不可（→ 2.8）。

**BQ側での実体化（2択）**
- **(a) メールキーの mapping 表**：`ref.principal_attributes(email, honbu_code, …)` を置き、RLSサブクエリで `honbu_code IN (SELECT honbu_code FROM ref.principal_attributes WHERE email = SESSION_USER())`。SAが叩けば `SESSION_USER()` はSAメールを返すので `(sa-xx@…iam.gserviceaccount.com,'1234')` で表現できる。
- **(b) Cloud Identity グループ所属**：本部ごとにグループを作り `GRANT TO ('group:…')`。属性は「行」でなく「メンバー」。**Entra認証なら Entra→Google Cloud Identity のグループ・プロビジョニングが追加で要る**ため、(a) の方が素直なことが多い。

**federation で逃げられない**：`EXTERNAL_QUERY` はオンプレに届かず、結果（一時テーブル）はサブクエリ型RLSに使えない（→ 2.8［31,2]）。**よって同期がほぼ必須**。唯一の回避は「BQで強制せずAPIに寄せ、APIが request-time にオンプレ/OpenGIMを引く」＝論点3(b)／論点4#1（ただし直BQ/sharing購読者は覆えない）。

**同期手段（オンプレ→BQ／マスタ系＝小・低頻度を前提）**

| 方式 | 中身 | 向く状況 | 鮮度 |
|---|---|---|---|
| **バッチ（推し）** | オンプレ抽出→GCS→**BQ定期ロード**（or GCS上ファイルを**BigLake外部表**＝RLSサブクエリ可）。Composer/Workflows/Scheduler＋Cloud Run で実行 | 組織マスタの小・低頻度 | 日次〜時次 |
| **マネージドCDC** | **Datastream→BQ**（Oracle/MySQL/PostgreSQL/SQL Server、オンプレ可、insert/update/delete＋バックフィル）。要 Cloud VPN/Interconnect［29,30,32］ | 分単位の鮮度が要る | 秒〜分 |
| Dataflow / 3rd party | Dataflow JDBC→BQ、Fivetran/Airbyte(OSS)/Striim/Debezium | 既存基盤・非対応ソース | 構成次第 |
| イベント駆動 | オンプレが変更イベント→Pub/Sub→BQ subscription | 即時反映 | 秒 |
| グループ経路((b)用) | オンプレ組織→Cloud Identity（AD/LDAPは GCDS、IdPは SCIM） | グループ型RLS | 方式次第 |

**落としどころ／推し**：組織・識別子マッピングは小さく低頻度なので、**「オンプレ抽出→GCS→BQ定期ロード（or BigLake外部表）」で日次/時次**が第一候補。`ref` データセットを統制対象としてRLSサブクエリの参照先にする。分単位が要件なら Datastream。

**注意（2点）**
1. **mapping 表は認可データ＝センシティブ**。サブクエリ型RLSでは付与先が対象表だけでなく**参照表にも `bigquery.tables.getData` を要する**［3］ので、**消費者が mapping を直接読める**（他人の本部コードが見える）。開示可否を判断し、嫌なら本部ごと `GRANT TO group` の**非サブクエリ形**／authorized view で包む。マスキングは**非サブクエリ型RLSとのみ併用可**［2］。`filteredDataViewer` を広域IAM付与しない［3,4］。
2. **鮮度＝不整合窓（論点7）**。バッチは最大1サイクル遅延、CDCは秒〜分。RLSサブクエリは課金・プルーニング非対応なので **mapping 表は小さく保つ**［2］。

### 論点7：整合性・同期・鮮度
**問い**：強制点が2つ＝伝播経路2本。職改時の反映窓をどこまで許すか。
**経路**：Casbin=watcher/adapter、OPA=bundle ポーリング、BQ=group 反映 / mapping 表リフレッシュ / DDL 再デプロイ。
**罠**：テーブルをコピーすると RLS/CLS もコピーされるが**コピー先は元と非同期（独立）**になりドリフト源［5］。
**落としどころ**：「API は即時、BQ は表更新まで遅延」等の不整合窓を SLO として明示し許容範囲を決める。

### 論点8：オーサリング/ガバナンス(PAP)・デプロイ結合・ドリフト検出
**問い**：正本ポリシーの置き場・版管理・監査・テストをどう設計するか（C8）。
**回答パターン**：正本を Git/カタログに（CODEOWNERS＝データオーナー＋セキュリティ）、変更は git commit で監査［18］。**モデル単位の成果物（policy-as-product）**としてカタログ登録。api/dataform が別リポなら契約bumpで協調。
**ドリフト検出**：harness-engineering-guide の「計算的センサー＋approved fixtures」を流用＝**〈被験者＋文脈→見えるべき行/列〉のゴールデン期待値を API と BQ の両方に CI で当て、差分が出たら落とす**。#2/#3 を採っても回帰防止に併用。

### 論点9：買うか作るか＆エンジン選定（メタ）
**問い**：統制プレーン製品を買うか、自前で作るか。作るならどのエンジンか。
**回答パターン**：
- **買う**：Immuta/Privacera。一発で「1脳→多DB」を得るが年 \$10万〜\$50万+級で研究用途には過剰［26］。Immuta は AND/OR 混在不可の指摘あり（要検証）［27］。
- **作る**：OPA/Cedar/Casbin＋BQネイティブ。研究スコープ（検証用公開API）はこちら。**Casbin 単独だと論点1の②③が苦しい**点が効く。SQL フィルタ生成のエコシステムは OPA が厚い［12〜15］。Cedar は静的解析が強み［22］。

---

## 4. 世の中の派閥マップ（要約）

| 派閥 | 代表 | 思想 | この件への含意 |
|---|---|---|---|
| 外部化PDP / Policy-as-Code | OPA, Cedar, Topaz, Axiomatics, AuthZEN | 認可をコード化し1 PDPに集約・脱結合 | パターン1/3/6。**OPA の部分評価**が単一脳×SQL の鍵 |
| データ中心セキュリティ | BQ RLS/CLS, Snowflake RAP, Databricks UC ABAC | 強制をデータ層へ。どのクライアントも一律 | パターン0/4。プッシュダウン阻害＝全表スキャンの罠［28］ |
| 統制プレーン製品 | Immuta, Privacera | 1ポリシー→各DBへ自動反映（secure view 等） | パターン2の「買う」版［25,26］ |
| 関係/Zanzibar | OpenFGA, SpiceDB | タプル＆グラフで per-object 権限 | 行・列フィルタ生成は不得手。本件は当面 ABAC 優位 |
| 軽量ライブラリ埋め込み | **PyCasbin**, casl, oso | アプリ内 in-process の手軽さ | 粗い/アプリ認可に最適。**行・列の単一化は不得手** |
| 仕様＋適合テスト/Harness | approved fixtures, fitness functions | runtime 非統一・spec＋テストで縛る | パターン5。どの構成でも回帰防止に併用 |

---

## 5. 推奨たたき台（1案）＆ 新規性の置き方

### 推奨構成：パターン4を土台に、API は #3 で薄く乗せ、#5 で縛る
1. **正本は OPA/Rego 1本**（論点1＝spec＋decision-shape を狙う／Casbin に拘らない理由は行フィルタ生成）。
2. **行=BQ RLS（subquery＋`SESSION_USER()`＋OpenGIM 由来 mapping 表）、列=BQ CLS（ポリシータグ）** をデータ層の権威境界に（論点0/3/5/6）。直SQL/sharing 購読者も覆える。
3. **API は同じ Rego を部分評価して WHERE を生成**し BQ クエリに付加（論点4#3）。API 経由は二重防御で脳は1つ。
4. **OpenGIM → mapping 表の同期**を中心運用課題として明示（論点6/7）。実体化は**オンプレ→GCS→BQ定期ロード（or BigLake外部表）**を第一候補、分単位が要れば Datastream。federationは不可。SA は ETL/システム間限定・`TRUE` フィルタ免除。
5. **approved fixtures を CI で API と BQ の両方に当てる**（論点8）＝ドリフトを殺す。
6. カタログ(PAP) に Rego＋mapping 表スキーマ＋ポリシータグ束を**モデル単位の成果物**として登録。

**トレードオフ（正直に）**：「Casbin で全部」より主張は強くなるが、Casbin の手軽さは捨てる。

### 新規性（研究性）の置き方
「RBAC＋ABAC＋ReBAC を組み合わせる」「OPA/Cedar を使う」は**2026 のコモディティ**で新規性にならない。既製技術は部品と位置づけ、新規性は次に置く。

1. **運用モデルの反転**：利用者/UC 単位の付与でなく、**データオーナーがモデルごとにポリシーを一度だけ書く**（都度申請・都度開発・職改対応を消す）。
2. **人事マスタ由来のゼロメンテ伝播**：EntraID 認証→OpenGIM 属性を判定時に評価し、**異動・職改が棚卸しゼロで自動反映**されることを実証。
3. **単一ポリシー・複数強制点の一貫性**：1つの Rego から **BQ（RLS/CLS）と API（PDP）の両強制を導く**。←ここに実装上の研究余地。

一文化：「既製の認可エンジンと BigQuery の RLS/CLS を部品として、“モデル単位に一度書いたポリシーを、人事マスタ由来の属性で評価し、BQ と API の両チャネルへ一貫適用する”運用フレームワークを確立し、職改・異動時の権限メンテをゼロにできることを部門費モデルで実証する」。

---

## 6. パイロット検証項目（部門費 / VC本 × ST部門費）

- [ ] ワンマート上の Authorized View ＋ RLS で「ST部門費の行サブセット」を VC本に提供できるか（複製マートなし）。
- [ ] BQ sharing の linked dataset 経由で、**RLS が購読者の identity / 所属グループに正しく効くか**（まずグループ粒度）。
- [ ] `SESSION_USER()` ＋ OpenGIM 由来 mapping 表の subquery RLS が機能するか／課金・プルーニング影響の実測。
- [ ] 列の出し分け（予算=全員／実績=管理職のみ）が CLS（ポリシータグ＋マスキング）で表現できるか。
- [ ] 同一 Rego の部分評価で、API の BQ クエリ用 WHERE が想定通り生成されるか。
- [ ] approved fixtures（被験者×文脈→見えるべき行/列）を API と BQ の両方に当て、差分ゼロを確認。
- [ ] BQ sharing を「使う/使わない（Authorized View＋IAM のみ）」の運用コスト比較。
- [ ] Subscriber Project と SA の発行・最小権限・鍵レス(WIF)・IaC 化の手順確立。
- [ ] オンプレにしか無い属性（SA/個人→本部コード・組織階層）の**同期パイプライン**（オンプレ→GCS→BQ定期ロード or BigLake外部表、必要なら Datastream）を構築し、`ref` データセットとして統制。
- [ ] mapping 表のプライバシー：サブクエリRLSで**参照表に `getData` が要る**ため消費者が mapping を読めてしまう点の可否確認（嫌なら非サブクエリ形／authorized view で包む）。
- [ ] federation（`EXTERNAL_QUERY`）はオンプレ非対応・RLSサブクエリ非対応である前提を確認し、同期方式と**鮮度SLO**（職改の反映窓）を決定。

---

## 付録A：レビュアー原コメント（原文）

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

## 付録B：出典

1. Google Cloud — Authorized views / BigQuery sharing. https://docs.cloud.google.com/bigquery/docs/authorized-views
2. Google Cloud — Introduction to row-level security（課金・Storage API 非互換・TRUE フィルタ）. https://docs.cloud.google.com/bigquery/docs/row-level-security-intro
3. Google Cloud — Use row-level security（SESSION_USER／lookup subquery）. https://docs.cloud.google.com/bigquery/docs/managing-row-level-security
4. Google Cloud — Best practices for row-level security（filteredDataViewer・組織内用途）. https://docs.cloud.google.com/bigquery/docs/best-practices-row-level-security
5. Google Cloud — Using row-level security with other features（copy 非同期・CLS 併用）. https://docs.cloud.google.com/bigquery/docs/using-row-level-security-with-features
6. classmethod / DevelopersIO — BigQuery Sharing permissions（linked dataset は bigquery.user のみ）. https://dev.classmethod.jp/en/articles/bigquery-sharing-permission-points/
7. Medium (N. Rajput) — BigQuery Row-Level Policies + Views（複製しない）. https://medium.com/@hadiyolworld007/bigquery-row-level-policies-views-tenant-isolated-analytics-without-duplicate-tables-f588ab41c3da
8. Medium / Google Cloud Community (S. Karadag) — Manage Access to Analytics Hub + Authorized Views（グループ粒度・プロジェクト構成）. https://medium.com/google-cloud/how-to-manage-access-bigquery-analytics-hub-and-authorized-views-e38b47e6a0b9
9. OneUptime — Create BigQuery Views and Authorized Views（別データセット・authorize・同一ロケーション）. https://oneuptime.com/blog/post/2026-02-17-how-to-create-bigquery-views-and-authorized-views-for-secure-data-sharing/view
10. Devoteam — 3 ways to protect BigQuery data with RLS（アプリ vs ウェアハウス・非プルーニング）. https://www.devoteam.com/expert-view/3-options-to-protect-bigquery-data-with-row-level-security/
11. Casbin — Data Permissions（implicit assignments / BatchEnforce は真偽配列）. https://www.casbin.org/docs/data-permissions/
12. Open Policy Agent — REST API Reference（Compile API・SQL/UCAST・Accept ヘッダ）. https://www.openpolicyagent.org/docs/rest-api
13. Open Policy Agent — Data filtering / Partial evaluation（unknowns→残余条件→SQL）. https://www.openpolicyagent.org/docs/filtering/partial-evaluation
14. Styra — Data Filters Compilation API（Enterprise OPA・列マスキング compile）. https://docs.styra.com/enterprise-opa/reference/api-reference/partial-evaluation-api
15. Permit.io — Data Filtering（Source-Level / Partial Evaluation・SQL 翻訳）. https://docs.permit.io/how-to/enforce-permissions/data-filtering/
16. AWS Prescriptive Guidance — PDP via OPA / PAP・PDP・PEP / Cedar・OPA. https://docs.aws.amazon.com/prescriptive-guidance/latest/saas-multitenant-api-access-authorization/opa.html
17. Axiomatics — Externalized Authorization (EAM). https://axiomatics.com/resources/reference-library/externalized-authorization
18. ssojet — Authorization as a Service and Policy-as-Code（PEP/PDP・git 監査）. https://ssojet.com/ciam-qna/authorization-as-a-service-and-policy-as-code
19. OpenFGA — Authorization Concepts（ReBAC は RBAC の上位集合・Conditions/Contextual Tuples）. https://openfga.dev/docs/authorization-concepts
20. Auth0 — Understanding ReBAC and ABAC Through OpenFGA and Cedar. https://auth0.com/blog/rebac-abac-openfga-cedar/
21. CIAM Compass — RBAC vs ABAC vs ReBAC（2026 ハイブリッド）/ FGA 実装ガイド. https://guptadeepak.com/ciam-compass/guides/rbac-vs-abac-vs-rebac/
22. StrongDM — Cedar Policy Language (CPL): 2026 Complete Guide. https://www.strongdm.com/cedar-policy-language
23. Permit.io — Top Open-Source Authorization Tools for Enterprises in 2026. https://www.permit.io/blog/top-open-source-authorization-tools-for-enterprises-in-2026
24. DEV (kanywst) — AuthZEN Authorization API 1.0 Deep Dive（OpenID, 2026-03）. https://dev.to/kanywst/authzen-authorization-api-10-deep-dive-the-standard-api-that-separates-authorization-decisions-1m2a
25. Immuta — Google BigQuery Integration Documentation 2026.1（policy push・1:1 secure view）. https://documentation.immuta.com/2026.1/configuration/integrations/google-bigquery
26. Modern DataTools — Immuta Review (2026)（単一統制プレーン・価格）. https://www.modern-datatools.com/tools/immuta
27. Privacera — Privacera vs Immuta（多プラットフォーム・AND/OR 制約の指摘＝要検証）. https://privacera.com/privacera-vs-immuta/
28. Databricks — Unity Catalog ABAC: requirements / performance / policies（比較対象）. https://docs.databricks.com/aws/en/data-governance/unity-catalog/abac/requirements
29. Google Cloud — Datastream for BigQuery（オンプレ含む Oracle/MySQL/PostgreSQL/SQL Server を CDC でBQへ直接）. https://cloud.google.com/datastream-for-bigquery
30. Google Cloud — Datastream overview（対応ソース・プライベート接続）. https://docs.cloud.google.com/datastream/docs/overview
31. Google Cloud — Introduction to federated queries（`EXTERNAL_QUERY` 対応は AlloyDB/Spanner/Cloud SQL のみ・結果は一時テーブル）. https://docs.cloud.google.com/bigquery/docs/federated-queries-intro
32. OneUptime — Replicate Oracle changes to BigQuery using Datastream（オンプレは VPN/Interconnect 必須）. https://oneuptime.com/blog/post/2026-02-17-replicate-oracle-database-changes-bigquery-datastream/view
