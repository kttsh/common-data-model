# FastAPI向け ABAC認可ライブラリ／ポリシーエンジン比較（OSS・行/列レベルフィルタリング対応）

> 対象ユースケース: FastAPI（確定）/ in-process（アプリ内組み込み）/ ABAC（自部門・役職などの属性）/ エンドポイント単位 + 行・列レベル制御の両方 / OSS限定

## 採用決定: PyCasbin

本プロジェクトでは **PyCasbin（Apache 2.0）** を認可ライブラリとして採用する。

**採用理由**

- **唯一の主要な純in-process Python選択肢**: アクティブにメンテされており、サイドカーや別サーバを立てずFastAPIプロセス内で完結する。in-process前提の要件に最も素直に合致する。
- **ABAC（自部門・役職）対応**: `eval()` matcherで subject属性を直接参照できる（例: `r.sub.Department == r.obj.Department && r.sub.Role == "manager"`）。
- **エンドポイント単位の認可が得意**: RESTfulモデルで subject/path/method をクリーンにマッピングできる。
- **FastAPI連携が本比較中で最も充実**: `fastapi-authz`, `fastapi-casbin-auth`, `casbin-fastapi-decorator` 等が利用可能。
- **ライセンスがクリーン**: Apache 2.0。BSL/AGPLの懸念なし。

**採用にあたって自前で実装する範囲（PyCasbinの非対応領域を手組みで補完）**

- **行レベルフィルタ**: PyCasbinはクエリ述語のプッシュダウンを生成しない。`GetImplicitPermissionsForUser` / バッチenforce で許可範囲を取得し、データ層（SQLAlchemy `where()`）で述語へ変換する。
- **列レベルマスキング**: ネイティブ非対応。属性に基づくPydanticのフィールド除外/値置換で実装する。
- **PIP（属性取得）**: 部門・役職などの属性をユーザモデル/JWT/オンプレSQL Serverから取得する層を用意する。

**見直しの閾値（将来OPAへの移行を検討する条件）**

- ABACの `eval()` matcher が概ね10〜15ルールを超える、またはmatcherのテスト・保守が困難になった場合。
- ポリシーの外部記述・監査・テスト容易性の要求が高まり、行レベルSQL/UCASTフィルタや列マスクのネイティブ生成（OPA≥v1.9.0のCompile API）が必要になった場合。
- その際はPyCasbinをエンドポイント認可に残しつつ、データフィルタ判定をOPA併設へ段階移行する構成が取れる。

## 結論（先出し）

- **全ハード要件（真の in-process Python + ABAC + エンドポイント単位 + 行レベルのフィルタ生成 + 列マスキング）を単独で満たすOSSエンジンは存在しない。** 最も近いのは **OPA**（OSSコアにデータフィルタリング/部分評価＋列マスキングを内蔵。ただしCompile APIは併設サーバ/サイドカー経由が自然で、純粋なPython組み込みではない）と **Casbin/PyCasbin**（真に in-process Python・matcherでABAC可。ただし行/列フィルタ生成は無く手組みになる）。
- **Oso OSS は非推奨化済み**（Python向け行レベル・データフィルタリングのベストだった組み込みライブラリ）、**Cedar は公式Pythonバインディング無し・SQL/list-objects機能も無し**（部分評価は実験的でRust限定）。**OpenFGA / Permify はサーバ型のReBACエンジン**（ABACは条件式で部分対応、行レベルはオブジェクト列挙であり述語プッシュダウンではない）で、in-process前提の要件に反する。
- この用途では現実的には **手組みの in-process Python PDP/PEP + ポリシー判定にOPAまたはCasbin、行/列フィルタはデータ層（SQLAlchemy）で実装** が定石。エンドポイント単位はどのライブラリも得意だが、行/列フィルタはどれを選んでもほぼ確実に独自グルーコードが要る。

## マスター比較表

| エンジン | ライセンス | メンテ／最新リリース | star(2026目安) | in-process(Py) | ABAC(部門/役職) | 行レベル＋方式 | 列レベル | ポリシー言語 | FastAPI連携 | 主な制約 |
|---|---|---|---|---|---|---|---|---|---|---|
| **OPA** (open-policy-agent/opa) | Apache 2.0 | アクティブ・v1.16.x系・高頻度 | 約11.8k | △ Goバイナリ／RESTサイドカー／WASM (`opa-wasm`/`opa-wasmtime`, MIT) | ○ Regoで`input`属性を評価 | **○** Compile API/部分評価→SQL or UCAST AST（OSSネイティブ。OPA≥v1.9.0 / EOPA≥v1.44.0 が必要） | **○** 同`/v1/compile/filters/{path}` APIの`masks`で列マスキング | Rego | `fastapi-opa`（OIDC+OPA）／compile呼出は自作 | WASMは部分評価/compile非対応・データフィルタはOPAサーバ併設が必要 |
| **Casbin / PyCasbin** | Apache 2.0 | アクティブ・PyCasbin v2.x (2025) | Go約17k+／py約1.6k | **○** 純Python | ○ `eval()` matcherでABAC・`r.sub.Department`,`r.sub.Role` | △ 述語生成は無し・`GetImplicitPermissions`/バッチenforce＋アプリ側 | × 手組み | PERMモデル(.conf)＋ポリシー(CSV/DB) | `fastapi-authz`, `fastapi-casbin-auth`, `casbin-fastapi-decorator` (MIT) | クエリ述語プッシュダウン無し・行/列は手動・ABAC matcherは規模拡大で煩雑化 |
| **Cedar** (cedar-policy/cedar) | Apache 2.0 | アクティブ(Rust)・Linux Foundation | 約1.5k | △ Rustクレート／3rdパーティ`cedarpy`のみ (Apache 2.0) | ○ RBAC+ABAC・context属性 | **×(標準)** 部分評価(TPE, RFC 0095)は実験的かつRust限定・SQL/list-objects無し | × | Cedar DSL | 公式無し・`cedarpy`上に自作 | 公式Pythonバインディング無し・partial-evalはPython未公開・データフィルタ機能無し |
| **OpenFGA** (openfga/openfga) | Apache 2.0 | アクティブ・CNCF Incubating (2025/10/28) | 約5k | × サーバ/サイドカー(Go)・Python `openfga-sdk` はクライアント | △ ReBAC主体・ABACはconditions(CEL)＋contextual tuplesで部分対応 | **○(列挙)** `ListObjects`/`StreamedListObjects`が許可IDを返す（SQL述語ではない） | × | FGA DSL＋JSONモデル | SDKのみ・公式FastAPIミドルウェア無し | サーバ型(in-process要件に反する)・属性はtuple/conditionでモデル化・ListObjectsはID列挙 |
| **Permify** | Apache 2.0 | アクティブ・FusionAuthが2025/11/20買収（OSSは継続） | 約5.6k | × サーバ(gRPC/REST)・Go | ○ schema DSLでRBAC/ReBAC/ABAC | △ `LookupEntity`が許可IDを返す | × | Permify schema DSL | `permify-python` クライアントSDK | サーバ型・述語プッシュダウン無し・列マスキング無し |
| **Oso (OSSライブラリ)** | Apache 2.0 | **2023/12/18 非推奨化**・重大バグ修正のみ・最新`oso` 0.27.3 | 約3.5k | **○** Pythonライブラリ(Rustコア) | ○ Polarで属性評価 | **○** データフィルタリング/`authorized_query`→SQLAlchemy/Djangoフィルタ（Python向けベスト） | フィールド単位 `authorize_field` | Polar | 公式FastAPIミドルウェア無し・ライブラリ利用可 | レガシーOSSはOso Cloudへ移行で非推奨・新規不適 |
| **py-abac** | Apache 2.0 | **停滞**（12ヶ月超リリース無し・Snyk "Inactive"） | 小 | ○ Pythonライブラリ | ○ XACML風JSONポリシー・PIP属性プロバイダ | × Permit/Deny後にアプリ側 | × | JSON (XACML風) | 無し | 未メンテ・Mongo/SQLバックエンド必要・データフィルタ無し |
| **Vakt** | Apache 2.0 | 低活動・v1.5.0 | 小 | ○ Pythonライブラリ | ○ ポリシー/属性ベース・IAM風 | × | × | Pythonオブジェクト/JSON | 無し | ニッチ・正規表現の性能注意・データフィルタ無し・コミュニティ小 |
| **Warrant** | Apache 2.0 (OSS) | **非推奨** — WorkOSが買収(2024/4)・WorkOS FGAへ | n/a | × サーバ | △ Zanzibar＋ポリシー属性式 | 列挙(Zanzibarクエリ) | × | Warrant型/ポリシー | n/a | OSSは実質放棄・WorkOS FGAへ移行 |

## 手組み（in-process Python ABAC PDP/PEP）

| 観点 | FastAPIでの手組み | ライブラリより優位なケース |
|---|---|---|
| **デプロイ** | 純in-process Python・追加サービスゼロ | in-process要件を常に満たす |
| **ABAC（自部門/役職）** | PDP関数が自前ユーザモデル/PIPから部門・役職を読む。自由に利用可 | 属性が既にDB/JWTにある場合 |
| **エンドポイント単位** | FastAPI `Depends()` / デコレータからPDP呼出 | 単純ケース・DI密結合 |
| **行レベル** | PEPがフィルタ構造体を返し→SQLAlchemy `where()` 述語へ変換 | ORMを自前管理し、エンジン無しで述語プッシュダウンしたい場合 |
| **列レベル** | 属性に応じたレスポンスモデルのフィールドマスキング（Pydantic） | そもそもネイティブ対応エンジンが少ない |
| **コスト** | ポリシー表現力・テスト・監査ログ・変更管理を自前で負う | ポリシー集合が小さく安定なら有利・複雑化/外部記述が必要だと不利 |

**構成要素のスケッチ**

- **PIP**（属性取得）: `get_user_attributes()` が部門・役職を返す
- **PDP**（判定）: `decide(subject, action, resource_type) -> Decision + filter_spec`
- **PEP**（強制）: 403を投げる or フィルタを注入するFastAPI依存
- **行フィルタ**: `filter_spec` → `Query.filter(...)` へ変換
- **列マスク**: 属性に基づくPydantic `model_dump` の除外/値置換

## 詳細

### OPA
Apache 2.0・約11.8k star・非常にアクティブ。Regoでエンドポイント単位（`allow`）と `input.subject.department` / `input.subject.role` のABACを表現。この用途で際立つのは **Compile API / 部分評価**：未知数(unknowns)を含むRegoポリシーを残差(residual)に変換し、**SQL WHERE句 または UCAST AST** として出力できる。OPA公式曰く「RegoデータフィルタポリシーのSQL/UCAST式への変換を使うには、新しいバージョンのOPA(≥v1.9.0)またはEOPA(≥v1.44.0)が必要」——このSQL/UCASTフィルタ生成（旧Styra Enterprise限定）はコアOSSに取り込まれた。**列マスキング** は同じ `/v1/compile/filters/{path}` compile APIで、レスポンスの `masks` オブジェクトを使う（例: `masks.tickets.description.replace.value := "<description>"`、特権ロールでは除去）。

注意点:
1. compile APIは稼働中のOPAインスタンス（サイドカー/併設デーモン）に対して使うのが最も自然——OPA公式もGo以外の言語ではOPAをホストレベルのデーモンとして配置することを推奨。
2. **WASMモード（`opa-wasm`,`opa-wasmtime`・いずれもMIT）は部分評価/compileに非対応**、直接の判定評価のみ——よって真のin-process Pythonにすると行フィルタ機能を失う。

ユーザは既にOPA/Conftestを使用しており、Rego習熟は利点。

### Casbin / PyCasbin
Apache 2.0。アクティブにメンテされる唯一の主要な**純in-process Python**選択肢。ABACは構造体属性を参照する `eval()` matcherで対応（例: `r.sub.Department == r.obj.Department && r.sub.Role == "manager"`）。エンドポイント/RESTfulモデルで「このユーザがこのエンドポイントを叩けるか」をクリーンに扱える（`fastapi-authz` がsubject/path/methodをマップ）。**行レベル**: 部分評価/述語生成は無く、ドキュメント上の「データ権限」手法は暗黙の権限照会（`GetImplicitPermissionsForUser`）やバッチenforceで、その後アプリ側でフィルタ——つまり述語は手組み。**列レベル**: ネイティブ無し。FastAPI連携は本比較中で最も充実。

### Cedar
Apache 2.0・約1.5k star・AWS/Linux FoundationがRustでメンテ。`context`/属性条件によるRBAC+ABAC。**公式Pythonバインディング無し**——唯一の選択肢は3rdパーティの **`cedarpy`**（Apache 2.0・最新`cedarpy 4.8.0`は2025/12/18・Cedarエンジン4.8.x追従）だが、公開APIは `is_authorized`, `validate_policies`, `format_policies` のみ。**部分評価** はRustクレートに実験的フラグ付きで存在（RFC 0095「Type-Aware Partial Evaluation」TPE、および旧 `partial-eval`）するが、明示的にunstable（「いかなるリリースでも破壊的変更の対象」）で、**どのPythonバインディングにも非公開**。Cedarに **ListObjectsやSQL述語生成は無く**、前方評価用の残差ポリシーを出すのみ。結論: cedarpy経由のエンドポイント/アクションABACは可能だが、行/列フィルタはPythonでは利用不可。

### OpenFGA
Apache 2.0・CNCF Incubating（CNCF曰く「2022/9/14受理、2025/10/28にIncubating昇格」）・約5k star・OktaとGrafanaがメンテ。アーキテクチャは**サーバ/サイドカー**(Go)で、Python `openfga-sdk` はネットワーククライアント——in-process前提の要件に反する。ReBAC主体。**ABAC**は部分対応で、**conditions**（Google CEL式。ドキュメント曰く「CEL式の評価コスト上限はデフォルトで100」）と **contextual tuples**（「リクエスト当たり現状100の上限」・CLIテストハーネスでは20件のより厳しい上限）が部門/役職/セッション文脈を運べる。**行レベル**は `ListObjects`/`StreamedListObjects` で許可IDのリストを返す（その後 `WHERE id IN (...)`）——述語プッシュダウンではなく列挙。列マスキングは無し。

### Permify
Apache 2.0・約5.6k star。**FusionAuthが買収**（「2025年11月20日、FusionAuthはPermifyの買収を発表」・「FusionAuth FGA by Permify」へリブランド）。コアはGitHub上でOSSのまま継続し、統合製品/価格は2026年予定。サーバ(gRPC/REST)・Go。schema DSLでRBAC/ReBAC/ABAC対応。行レベルは `LookupEntity`（許可ID）。サーバ型→in-process要件に反する。列マスキング無し。

### Oso (OSSライブラリ)
Apache 2.0。**レガシーのオープンソースOsoライブラリは2023/12/18に非推奨化**（OsoはOso Cloudへ転換）。告知曰く「ライブラリをEOLにはせず、サポートと重大バグ修正は継続する」——最新PyPIリリースは `oso 0.27.3`。新規には非推奨。技術的には**Python in-processの行レベルとしてはベスト**だった: Polarポリシー＋データフィルタリング（`authorized_query`/`authorized_resources`）が制約を**SQLAlchemy/Django ORMフィルタ**に変換（述語プッシュダウン）、加えてフィールド単位の `authorize_field`。非推奨でなければこの用途のトップ候補だが、新規の認可コアに非推奨ソフトを採用するのはリスク。

### py-abac / Vakt
いずれもApache 2.0・純Python・XACML/IAM風ABAC＋PIP属性プロバイダ——要求されたPDP/PEP/PIPアーキテクチャと概念的に合致。しかし **py-abacは停滞**（1年超リリース無し・Snykのメンテ評価は"Inactive"）、**Vaktも低活動**でコミュニティ小。どちらも行レベル述語生成や列マスキング無し、外部ストレージバックエンドが必要。参照アーキテクチャとしては有用だが依存先としてはリスク。

### Warrant
WorkOSが買収(2024/4)・OSSは非推奨化されホスト型WorkOS FGAへ誘導。候補外。

## 推奨（意思決定向けステージ別）

1. **「in-process Python」に最も寄せ、新規インフラを最小化したい → エンドポイント+アクションABACはCasbin/PyCasbin、行/列フィルタはSQLAlchemyで手組み。** 見直しの閾値: ABACの `eval()` matcherが約10〜15ルールを超える、またはテストしづらくなったら、ポリシーをOPAへ移す。
2. **外部化された・テスト可能な・監査可能なポリシーを重視し、既にOPA/Conftestを使っている → OPAを併設（サイドカー/デーモン）し、OSSのCompile API（OPA≥v1.9.0）で行レベルSQL/UCASTフィルタ＋列マスクを生成。** 本比較中、行フィルタと列マスクの双方をOSSでネイティブ生成できる唯一のエンジン。「in-process」が「co-located process（併設プロセス）」になること、WASM組み込みモードでは部分評価ができないことを許容する必要がある。
3. **行/列のデータフィルタは、どのエンジンでも変換層を自前で持つ前提で計画する。** allow/deny＋filter-specの判定にエンジンを使い、述語プッシュダウンとフィールドマスキングはデータ/シリアライズ層（SQLAlchemy `where()`、Pydanticフィールド除外）で実装。
4. **新規構築で避けるもの:** Oso OSS（非推奨）、Cedar-for-Pythonの行フィルタ（利用不可）、py-abac/Vakt（未メンテ/低活動）、Warrant（消滅）。OpenFGA/Permifyは別途認可サーバとReBAC的モデリングを許容できる場合のみ。

**推奨を覆す材料**: TPEを公開する公式かつメンテされたCedar Pythonバインディングが出ればCedarは強力なin-process ABAC+行フィルタ候補になる。復活/再ライセンスされた次世代Oso OSSが出ればPythonデータフィルタのエルゴノミクスが最良に戻る。

## 注意点

- 「in-process」が最難の制約: 真の純Python in-processは **Casbin, Oso(OSS), py-abac, Vakt** のみ。OPA/Cedarは他言語(Go/Rust)かWASM経由でしかin-processにならず（WASMは部分評価を失う）。OpenFGA/Permify/Warrantはサーバ。
- OPAのOSSネイティブなデータフィルタ(SQL/UCAST)と列マスキングはStyra Enterprise OPAからコアOPAへ移植されたもの。SQL/UCASTフィルタはOPA≥v1.9.0 (EOPA≥v1.44.0)が必要。デプロイするOPAのバージョンが `/v1/compile/filters/...` と `masks` オブジェクトに対応しているか確認すること。
- star数は2026年のGitHub概数（GitHubの省略表記。キャッシュページにより差異あり。例: OpenFGAは4.4k〜5.7k、Cedarは約1.5kだがstargazerの綺麗な数値ではなくcommit-activity由来）。レンジとして扱うこと。
- 推奨セットのプロジェクトは全てApache 2.0。BSL/AGPLは使っていない。Permify・OpenFGAは商用支援下でもApache 2.0。FusionAuthの主力アプリはsource-availableだが、PermifyコアはApache 2.0のまま。
- Cedarの部分評価機能はRustクレートで明示的にexperimental/unstableとされ、いかなるリリースでも変更され得る。現時点でPythonからは呼び出せない。
