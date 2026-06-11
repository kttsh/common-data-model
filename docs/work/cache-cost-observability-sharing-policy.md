# キャッシュ・コスト管理・Observability・BigQuery Sharing 方針案

> 作成日: 2026-06-11
> ステータス: 討議用ドラフト(方針案)
> Related: `../02-architecture/platform-architecture-decision.md`(案A 確定)、`../03-authorization/row-level-filtering-layering.md`、`../02-architecture/bigquery-dataform-design-rules.md` §6-6
> 情報時点: 2026年6月(Web調査込み)

---

## 0. 前提

案A(APIM 組込みキャッシュ + プロセス内 LRU + 単一 SA で BigQuery アクセス)の確定事項を変更しない。本書は案Aの上に載せる運用方針を定める。

単一 SA 構成の帰結として、**BigQuery から見える「人」は SA ただ1つ**であり、エンドユーザー単位の制御・計測は BigQuery ネイティブ機構では成立しない。コスト管理・Observability の方針はすべてこの制約から導出される。

---

## 1. キャッシュ戦略

### 1.1 2026年時点の前提更新

- **BI Engine は廃止されていない**(存続、認可ビュー・CLS・データマスキングとも併用可)。ただし 2025年9月〜2026年初頭にかけて **advanced runtime(拡張ベクトル化・short query optimizations・history-based optimizations)が全プロジェクトのデフォルトランタイムとして展開され GA** となった。無料・コード変更不要・自動適用。
- つまり「BI Engine 予約(有償)を買う前に、無料の自動高速化レイヤが既に効いている」が現在の出発点。BI Engine を案Bの再検討トリガー扱いにした判断は据え置きでよい。
- BigQuery の**クエリ結果キャッシュ**(無料・自動・約24時間)は「同一クエリテキスト + 同一パラメータ + 対象テーブル未更新」が条件で、**プリンシパル単位**で保持される。単一 SA 構成では全エンドユーザーのキャッシュが同一プリンシパル下で共有されるため、**認可パラメータのカーディナリティが低い場合(部署単位フィルタ等)は別ユーザー間でもヒットする**。日次更新の Gold に対しては実質的な無料キャッシュ層として機能する。

### 1.2 キャッシュスタック(方針案)

| # | 層 | 機構 | 費用 | 対象 | TTL/鮮度の決め方 |
|---|---|---|---|---|---|
| 1 | クライアント | `Cache-Control` / `ETag` を API レスポンスに付与 | 無料 | 共通参照系 | データ更新間隔から導出 |
| 2 | APIM 組込み | `cache-store` ポリシー | 無料 | 共通参照系(マスタ・スキーマ・公開メタデータ)のみ。per-user 系は対象外 | 秒〜分(揮発性のため短く) |
| 3 | プロセス内 LRU | cachetools | 無料 | Open-GIM 属性(30〜60秒)、マスタ、コンパイル済みポリシー | 属性は短 TTL 固定 |
| 4 | BQ 結果キャッシュ | 自動 | 無料 | 同一クエリ+同一パラメータの再実行 | 自動(テーブル更新で無効化) |
| 5 | advanced runtime | 自動(デフォルト) | 無料 | 全クエリ | 設定不要 |
| 6 | マテビュー | Dataform で個別定義 | ストレージ+リフレッシュ | 頻出統合モデルのみ個別判断 | 自動リフレッシュ |
| 7 | BI Engine | 予約(有償) | メモリ予約課金 | **保留**(再検討トリガー: SLO 恒常未達 かつ 1〜6 で改善不能) | — |

### 1.3 TTL 設計原則

- **TTL の入力はデータ更新頻度 = Dataform の実行スケジュール**。Gold テーブルが日次更新なら、論理上限は「次回更新時刻まで」。実際の APIM/プロセス内 TTL は揮発性リスクを踏まえ短め(秒〜分)に設定し、長い鮮度許容はクライアント側 `Cache-Control` に持たせる。
- エンドポイントを「共通参照系」「per-user 系」の2分類でタグ付けし、キャッシュ可否を OpenAPI 定義のメタデータとして宣言する(キャッシュ設定の散在を防ぐ)。
- per-user 系は共有キャッシュ禁止(R2 踏襲)。キャッシュするなら認可パラメータをキーに含めること。

---

## 2. コスト管理

### 2.1 設問への直接回答: 「プロジェクト × データセット × 人 × 一定期間」は組めるか

| 軸 | ネイティブ可否 | 機構 |
|---|---|---|
| プロジェクト単位 / 日 | **○** | カスタムクォータ `QueryUsagePerDay`(既定 200 TiB/日、TiB 単位で引き下げ可) |
| 人(プリンシパル)単位 / 日 | **△** | `QueryUsagePerUserPerDay`。ただし**プロジェクト内の全ユーザー・SA に一律適用**で、特定ユーザーだけ別の値にはできない |
| データセット単位 | **×** | ネイティブ機構なし |
| 任意期間 | **×** | 日次のみ(毎日太平洋時間0時リセット)。週次・月次は不可 |

重要な制約が3点:

1. カスタムクォータは**オンデマンド課金のみ**に適用される。Editions(容量課金・予約)を使う場合はクォータではなく **max slots(オートスケール上限)** がコスト制御手段になる。現行課金モデルの確認が前提(→ §5 論点 C-1)。
2. クォータは**事前(proactive)に効く**ハードリミットで、超過クエリはエラーで拒否される。プロジェクトクォータが尽きると**そのプロジェクトの全員が止まる**。
3. 単一 SA 構成では `QueryUsagePerUserPerDay` = **SA 1本に対する上限**であり、エンドユーザー単位の制御にはならない。

### 2.2 多層防御(方針案)

| # | 粒度 | 機構 | 性質 | 実装場所 |
|---|---|---|---|---|
| 1 | クエリ単位 | `maximum_bytes_billed` を API が**全クエリに必ず設定**(必要に応じ dry-run で事前見積) | ハード | FastAPI(BigQuery クライアント設定) |
| 2 | SA / 日 | `QueryUsagePerUserPerDay` | ハード | GCP(Terraform 管理外。Service Usage API / Console) |
| 3 | プロジェクト / 日 | `QueryUsagePerDay` | ハード | GCP |
| 4 | エンドユーザー単位 | (a) APIM `quota-by-key` / `rate-limit-by-key`(JWT sub をキーに呼出数・帯域を制御) (b) スキャン量ベースはジョブラベル集計によるソフトリミット(下記) | ハード(呼出数)/ソフト(バイト) | APIM / FastAPI |
| 5 | データセット × 利用者の帰属 | **ジョブラベル**(`end_user`(ハッシュ)・`endpoint`・`entity`)を全ジョブに付与 → `INFORMATION_SCHEMA.JOBS` で日次集計 | 計測+ソフトリミット | FastAPI + 集計バッチ |
| 6 | 事後検知 | Cloud Billing 予算アラート + 課金データの BigQuery エクスポート | リアクティブ | GCP |

「データセット単位のハードリミット」が必要になった場合の代替は2つ:
**(a)** ジョブラベル集計ベースのソフトリミット(API 層で 429 を返す)、
**(b)** データセットをプロジェクトに分離する(BigQuery Sharing で BU 別プロジェクトに配布すれば、購読側のプロジェクトクォータが事実上「データセット × 組織」のハードリミットになる。→ §4.2)。

---

## 3. Observability

### 3.1 APIM 側でメトリクスは取れるか → **取れる(こちらを正とする)**

| 手段 | 内容 | 用途・注意 |
|---|---|---|
| Azure Monitor プラットフォームメトリクス | Requests・Capacity 等を**1分粒度**で自動発行 | アラート・容量判断 |
| 診断設定 → GatewayLogs を Log Analytics へ | リクエスト単位のログ(API・operation・subscription・呼出元・応答コード・レイテンシ) | **利用メトリクス集計の本命**。KQL で「誰が・どの API を・どれだけ」を出せる |
| Application Insights 連携 | E2E トレース・依存関係 | **サンプリング前提のため監査・課金集計には使わない**(統計分析用) |
| `emit-metric` ポリシー | ポリシーからカスタムメトリクス発行(次元付き) | 必要時のみ |
| 組込み分析(classic) | ポータルのダッシュボード | **classic 階層の組込み分析ダッシュボードは 2027年3月に廃止予定**。Azure Monitor ベースに寄せておく |

注意: 共有 APIM のため、**診断設定の変更権限と Log Analytics の送付先ワークスペース**は既存運用ルール(未確定事項 U3)に依存する。ここが本トピックの最大の社内確認事項。

### 3.2 BigQuery 側のメトリクス

| 手段 | 内容 |
|---|---|
| `INFORMATION_SCHEMA.JOBS(_BY_PROJECT)` | ジョブ単位の bytes billed・スロット消費・実行時間。**ジョブラベルで集計軸を切れる** |
| Cloud Audit Logs(Data Access) | 誰が(=SA が)何にアクセスしたかの監査証跡 |
| Cloud Monitoring | スロット使用率・スキャンバイト等のサービスメトリクス |
| 課金エクスポート → BigQuery | 請求ベースの事後分析 |

### 3.3 方針案: 相関 ID による3点測量

- **APIM(GatewayLogs)= 「誰が・どの API を・何回」の正本**。
- **FastAPI 監査ログ(Log Analytics)= 「どの認可条件(WHERE)が適用されたか」の正本**(確定済み方針の踏襲)。
- **BigQuery(JOBS + ジョブラベル)= 「各呼出が何バイト・何スロット消費したか」の正本**。
- 突合キーは API が払い出す `request_id`。FastAPI が BigQuery ジョブラベルに `request_id`・`end_user`(ハッシュ)・`entity` を載せることで、APIM ログ ↔ 監査ログ ↔ BQ ジョブが1本に繋がる。コスト管理 §2.2 の #4・#5 はこの仕掛けの上に成立するため、**ジョブラベル付与は API 実装の必須要件(ハーネスの computational sensor 対象)に格上げする**。

---

## 4. BigQuery Sharing

### 4.1 現状整理(2026年6月)

- **Analytics Hub は 2025年4月に「BigQuery sharing」へ改称**。コンソール表記は「Sharing (Analytics Hub)」。機能は連続している。
- 概念: **data exchange**(リスティングの入れ物)→ **listing**(共有データセットへの参照+メタデータ)→ 購読すると購読側プロジェクトに **linked dataset**(読み取り専用のポインタ。**データコピーなし**)が作られる。購読は **subscription** リソースとして管理され、**公開側は購読者一覧と利用メトリクスを参照できる**。
- 課金: ストレージは公開側、**クエリのコンピュートは購読側プロジェクトに課金**される。
- 制約・注意: 同一リージョン前提(必要なら linked dataset replica)。VPC Service Controls 境界内での公開は ingress/egress ルール整備が必要。egress 制御(コピー・エクスポート禁止)を listing 単位で設定可能。

### 4.2 本PJでの適用ポイント(方針案)

| 経路 | Sharing の要否 | 理由 |
|---|---|---|
| API 経路(単一 SA) | **不要** | SA には認可ビュー/認可データセットで直接権限を与える方が単純。Sharing を挟む利点がない |
| 直接参照経路(アナリスト/BI が BigQuery を直接叩く) | **適用候補(本命)** | 認可ビューの個別 IAM 配布より、listing 化により購読管理・棚卸し・利用メトリクスが構造的に手に入る |
| BU 別コスト分離 | **適用候補** | BU が自プロジェクトから購読 → スキャン課金が BU 側に発生し、BU プロジェクトのクォータがそのまま効く。§2.2 の「データセット × 組織のハードリミット」代替を兼ねる |

環境3面では **prod の `outputs`(Gold)のみを listing 化**する。dev/stg の共有は原則しない。

### 4.3 試行チェックリスト(「ためしてみた」用)

1. `analyticshub.googleapis.com` API 有効化、公開側プロジェクトに data exchange 作成(private)
2. `outputs` データセットを listing 化(説明・連絡先メタデータ込み)
3. 別プロジェクトの別アカウントに Analytics Hub Subscriber ロールを付与し購読 → linked dataset 生成を確認
4. linked dataset 越しのクエリで、課金が**購読側プロジェクト**に立つことを確認
5. 公開側で購読者一覧・利用メトリクスの見え方を確認
6. egress 制御(コピー/エクスポート禁止)を有効化して挙動を確認
7. `INFORMATION_SCHEMA.SCHEMATA_LINKS` で linked dataset の棚卸しが可能なことを確認

---

## 5. 明確にするべきポイント(討議用)

| # | 領域 | 論点 | なぜ今決めるべきか |
|---|---|---|---|
| A-1 | キャッシュ | 各 Gold テーブルの更新スケジュール(Dataform 実行頻度)の確定 | TTL 設計の唯一の入力。これが無いと TTL は決められない |
| A-2 | キャッシュ | メモの「BI engine」は案B要素の再開提案か、BigQuery 側キャッシュ機構の総称か | 前者なら再検討トリガー条件(SLO 恒常未達)の充足確認が先。後者なら §1.2 の無料スタックで足りる |
| C-1 | コスト | 現行 BigQuery の課金モデル(オンデマンド or Editions/予約) | **カスタムクォータはオンデマンド限定**。Editions なら制御手段が max slots に変わり §2 の前提が変わる |
| C-2 | コスト | 「人単位」の意図はエンドユーザーか SA か | エンドユーザーなら BigQuery では不可能で、API 層実装(APIM quota-by-key + ジョブラベル)が必須になる |
| C-3 | コスト | 上限到達時の挙動合意(API が 429 を返す / 縮退応答 / 管理者承認で解除) | クォータ枯渇時は**プロジェクト全員が止まる**ため、利用者への見せ方を先に合意しておく必要がある |
| C-4 | コスト | 期間は日次で足りるか | ネイティブは日次のみ。月次予算制御はソフトリミット(ジョブラベル集計)でしか組めない |
| O-1 | Observability | 共有 APIM の診断設定変更権限と Log Analytics 送付先(U3 の具体化) | APIM を利用メトリクスの正本にできるかはここで決まる。不可なら正本が FastAPI 監査ログに繰り下がる |
| O-2 | Observability | Datadog 併用の承認状況 | Log Analytics 一本で開始するか、二重送信の設計を最初から入れるか |
| S-1 | Sharing | API 非経由で BigQuery を直接参照する利用者が実在するか | 不在なら Sharing の適用範囲は「BU 別コスト分離」のみに縮小する |
| S-2 | Sharing | 購読側となる BU 側 GCP プロジェクトの有無・払い出しルール | linked dataset は購読側プロジェクトが無いと作れない。コスト分離案の成立条件 |
| S-3 | Sharing | Organization / VPC-SC の制約有無 | VPC-SC 境界内の listing は ingress/egress ルール整備が必要 |

---

## 参考(出典)

- BigQuery advanced runtime(2025/9〜2026年初頭に全プロジェクトのデフォルト化): https://cloud.google.com/bigquery/docs/advanced-runtime
- advanced runtime / fluid scaling GA(2026/4 Google Cloud blog): https://cloud.google.com/blog/products/data-analytics/unveiling-new-bigquery-capabilities-for-the-agentic-era
- BI Engine 概要(存続・認可ビュー/CLS と併用可): https://docs.cloud.google.com/bigquery/docs/bi-engine-intro
- カスタムクォータ(QueryUsagePerDay / QueryUsagePerUserPerDay、TiB 単位、日次、オンデマンド限定、個別ユーザー指定不可): https://docs.cloud.google.com/bigquery/docs/custom-quotas
- カスタムクォータ実践(2026/2): https://oneuptime.com/blog/post/2026-02-17-how-to-control-bigquery-costs-with-custom-daily-query-quotas/view
- APIM の Azure Monitor 監視(1分粒度メトリクス・GatewayLogs・診断設定): https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-azure-monitor
- APIM × Application Insights(サンプリング前提・監査用途不適): https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-app-insights
- APIM classic 組込み分析ダッシュボードの 2027/3 廃止: https://learn.microsoft.com/en-us/azure/api-management/monitor-api-management
- BigQuery sharing(旧 Analytics Hub)概要・linked dataset・購読管理・利用メトリクス: https://docs.cloud.google.com/bigquery/docs/analytics-hub-introduction
- 改称アナウンス(2025/4): https://medium.com/codex/google-just-announced-bigquery-sharing-78bb5dcbb9a0
- listing 管理・SCHEMATA_LINKS・VPC-SC 注意: https://docs.cloud.google.com/bigquery/docs/analytics-hub-manage-listings
