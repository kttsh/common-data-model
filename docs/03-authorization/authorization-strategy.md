# Authorization Strategy

> ステータス: 討議用ドラフト（論点出し＋技術方針）
> Sources: role memo, `../01-requirements/product-requirements.md`, `../02-architecture/platform-architecture-decision.md`
> 用途: チーム内ディスカッション（API・データモデルの権限管理）
> 前提: データソースは BigQuery、実行基盤は FastAPI + App Service / Functions、DAB は不採用方針
> Note: FastAPI は本メモ作成時点の実装仮説。ランタイム最終選定は `../04-research/api-runtime-framework-comparison-2026.md` を踏まえて別途確定する。
> 用語: **OpenGIM = 社内ユーザーリポジトリ**（役職・所属など認可属性の正本。Entra ID には当該属性が無いため別途参照する）

---

## 略語・用語

| 略語 | 正式名 | 説明 |
|---|---|---|
| DAB | Data API Builder | Microsoft の API 自動生成ツール（本件は不採用） |
| ABAC | Attribute-Based Access Control | 属性ベースアクセス制御 |
| RLS | Row-Level Security | 行レベルセキュリティ |
| CLS | Column-Level Security | 列レベルセキュリティ |
| JWT | JSON Web Token | 署名付きトークン |
| OIDC | OpenID Connect | OAuth 2.0 上の認証レイヤ |
| OBO | On-Behalf-Of | 代理（人の文脈を引き継いで実行） |
| RFC | Request for Comments | IETF の標準仕様文書 |
| OPA | Open Policy Agent | ポリシーエンジン |
| Cedar | （製品名） | AWS 由来のポリシー言語／エンジン |
| SA | Service Account | サービスアカウント |
| SLO | Service Level Objective | サービスレベル目標 |
| LRU | Least Recently Used | 古いものから捨てるキャッシュ方式 |
| TTL | Time To Live | キャッシュ等の有効期間 |

---

## Part 1. 叩き台への論点・指摘

「モデル単位の権限制御」という方向性自体は妥当。FIX させる前に潰しておきたい論点を重要度順に並べる。

### ① 【最重要】Entra ID だけで属性が揃う前提が崩れている

- 叩き台②は「Entra ID から人事情報（役職・所属）を取得」「Entra ID は最新人事と紐づくため棚卸不要」としている。
- 実際の Entra ID が持つのは ID・メール・せいぜいグループ程度で、原価センタ・職位・事業単位などの認可に必要な粒度は入っていない。
- `docs/01-requirements/product-requirements.md` でも「ユーザー属性はオンプレ SQL Server に実装・運用」と明記。
- **実態**: ID（認証）は Entra ID、認可属性は社内ユーザーリポジトリ（OpenGIM）を実行時に別途参照する二段構え。
- **影響**: 「棚卸不要」の根拠が変わる。鮮度は Entra ID ではなく OpenGIM の鮮度に依存する。
- → 明日の最大論点。

### ② OpenGIM（社内ユーザーリポジトリ）の参照方式・正本範囲を確定する

- **確定事項**: OpenGIM ＝ 社内ユーザーリポジトリ。役職・所属など認可に使う属性の正本はここ。叩き台②の「Entra ID から人事情報を取得」は、正しくは「Entra ID で認証 → OpenGIM から属性取得」。
- **残る確認事項**:
  - OpenGIM と `docs/01-requirements/product-requirements.md` が言う「オンプレ SQL Server（ExpressRoute 接続）」は同一か別物か。同一なら接続経路はそのまま、別物なら OpenGIM への到達経路（プロトコル・認証・閉域到達性）を定義する。
  - OpenGIM が保持する属性項目（役職レベルの区分、所属＝会社/部署/グループ、原価センタ、居住地など）の一覧と、各項目のキー（Entra ID の oid / UPN など何で突合するか）。
- これが認可設計（ABAC のポリシー入力）の前提になる。

### ③ 「SQL ライク（WHERE / SELECT）条件設定」は設計リスクが大きい

- 叩き台①は権限条件を WHERE / SELECT で記述する案で、参照に DAB を挙げている。
- (a) 参照先の DAB は今回不採用（`../02-architecture/data-api-builder-assessment.md` 参照）。
- (b) モデルオーナーが生 SQL 風の述語を自由記述する形は、インジェクション・正しさの検証・レビュー・変更履歴管理が難しい。
- (c) 「SELECT による列制御」の実体は列の許可/拒否リスト。自由 SQL ではなく統制された記法（ポリシー定義フォーマット）にすべき。
- → `docs/02-architecture/platform-architecture-decision.md` の D2（OPA / Cedar 等のポリシーエンジン比較）と接続させる。

### ④ 「申請なし・全社デフォルト権限」と監査・最小権限の緊張

- 要件「全社ユーザが申請なしでアクセス」と③のデフォルト権限は、`docs/01-requirements/product-requirements.md` の「監査ログ必須」「原価・単価などセンシティブ列の制御」とそのままでは両立しない。
- 叩き台自身の人事モデル例（人事部門以外は個人情報項目を閲覧不可）が示す通り、答えは「入口は全社開放、ただし行・列はモデル別ポリシーで常時適用」。
- **言い換え**: 「デフォルト＝開放」ではなく「デフォルト＝開放だが行列制御は常時適用」。そうしないと過剰権限付与に読める。

### ⑤ サービスアカウント（T ユーザー）と AI エージェントの扱いが矛盾しうる

- 叩き台は「システムからのアクセスは T ユーザー」とする一方、具体例では AI エージェントが「所属部署のデータのみ」に自動制限されている。
- 後者は人間の文脈を引き継ぐ（On-Behalf-Of）動きで、「T ユーザー固定権限」とは別物。
- エージェント経由が OBO なのかサービス ID 固定なのかを切り分けて定義する。
- `docs/01-requirements/product-requirements.md` でも SA 属性管理は未確定（L7）。明日詰める価値あり。

### ⑥ 「職改時メンテ不要」は条件付き

- ルールを属性参照で書けば人事異動には自動追従するが、組織改編で原価センタ体系・部署コード体系そのものが再編されるとルール側の見直しは発生する。
- 「人事異動には不要、組織再編時は要見直し」と精度を上げて記述する。

### ⑦ 行レベルの複数粒度が例で表現しきれていない

- `docs/01-requirements/product-requirements.md` は行レベルを「起票者単位〜事業単位の複数粒度」とするが、叩き台の例は部署単位中心。
- 起票者／部署／事業／全社のどれをどう記述するか、「主席以上」「部長以上」のしきい値をどこで定義するかをルール記法の設計に含める。

### ⑧ 監査の「どの条件で参照したか」を残す設計が抜けている

- 動的に WHERE を注入する方式なら、実際に適用したフィルタ条件を監査ログに残すのがガバナンスの肝。
- 叩き台には監査の記述がない。方針に一文入れる。

### ⑨ 属性参照とキャッシュ・レイテンシの接続

- 毎リクエストで社内リポジトリを引くとレイテンシ・負荷が増える（`docs/02-architecture/platform-architecture-decision.md` R6）。
- 属性キャッシュの TTL を認可設計とセットで決めないと、100〜500ms 目標と衝突する。

---

## Next Discussion Points

1. **①Entra ID 属性問題**: 認証は Entra ID、認可属性は OpenGIM（社内ユーザーリポジトリ）という二段構えで合意できるか。OpenGIM の参照方式・属性項目・突合キーの確定（②）。
2. **③SQL ライク条件の統制方法**: 自由 SQL ではなく、レビュー可能なポリシー定義フォーマット（ABAC / OPA / Cedar）に倒せるか。
3. **⑤T ユーザー／AI エージェントの認可**: OBO かサービス ID 固定かの切り分け。
4. **④デフォルト権限の言い換え**: 「入口開放＋行列制御常時適用」で合意。
5. ランタイムは App Service 主・Functions 補助で確定（異論があればここで）。

---
