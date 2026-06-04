# Platform Architecture Decision

> 作成日: 2026-05-26
> ステータス: **案A 採用決定**
> Related: `../01-requirements/product-requirements.md`

---

## 1. 決定事項サマリ

| 項目 | 決定 |
|---|---|
| **採用アーキテクチャ** | **案A: 最小構成（App Service + APIM 組込みキャッシュ + プロセス内 LRU + BigQuery ビュー）** |
| バックエンドランタイム | Azure App Service（Linux 想定） |
| **言語・フレームワーク** | **Python / FastAPI**（D1 確定 → `runtime-framework-decision.md`） |
| API ゲートウェイ | 既存共有 Azure API Management |
| キャッシュ | APIM 組込み（内部）キャッシュ + App Service プロセス内 LRU |
| データソース | BigQuery（Proxy 経由アクセス） |
| 認証 | Entra ID（OIDC + JWT、SA は Client Credentials / Managed Identity） |
| 認可 | App Service 内で ABAC（行レベル: SQL WHERE 注入、列レベル: 整形時マスク） |
| ユーザー属性参照 | Open-GIM（社内ユーザーリポジトリ。認可属性の正本。実体がオンプレ SQL Server と同一かは未確定論点。到達経路は ExpressRoute 想定） |
| 監査ログ | Azure Log Analytics（Datadog 併設可） |

---

## 2. 前提と制約

### 2.1 Requirements baseline

- データソース: GCP / BigQuery
- API 実行層: 社内 VPC 内 Azure
- フロント: 既存共有の Azure API Management
- 外部通信: Proxy 経由のみ（Azure → BigQuery も Proxy 経由）
- Azure ↔ オンプレ SQL Server: ExpressRoute
- 認証: Entra ID（OAuth 2.0 / OIDC）
- レスポンス目標: 100〜500ms（キャッシュヒット時、ミス時の許容上限は別途定義）
- 行・列レベル制御の実装場所: サービス層（API バックエンド）を第一候補

### 2.2 本検討で追加された制約

| # | 制約 | 影響 |
|---|---|---|
| C1 | **バックエンドランタイムは Azure App Service または Functions のみ**（Container Apps / AKS 不可） | メイン API は App Service 一択。Functions は補助的に併用候補 |
| C2 | **Azure Cache for Redis / Azure Managed Redis は利用不可（社内承認上の制約）** | APIM の外部キャッシュ機構が使えず、組込みキャッシュ＋プロセス内キャッシュに限定 |
| C3 | **Azure 側永続ストアは Azure SQL のみ承認済**。Cosmos DB / PostgreSQL Flexible Server 等は不可 | 案C（Azure ホットストア型）を将来採用する場合の選択肢が Azure SQL に限定される |

### 2.3 不明事項（要追跡）

| # | 不明点 | 影響 |
|---|---|---|
| U1 | App Service のプラン制約（Basic/Standard/Premium、Linux/Windows） | プロセス内キャッシュ容量、スケール戦略、コールドスタート挙動に影響 |
| U2 | Proxy 仕様（認証方式・対応プロトコル・BigQuery クライアント疎通可否） | 案A 成立性の根幹。設計初期で実機検証必須 |
| U3 | 既存 APIM の Subscription/Product 運用ルール | マルチテナント設計の前提 |
| U4 | Open-GIM（社内ユーザーリポジトリ）の認可属性スキーマ。実体がオンプレ SQL Server と同一かを含め確認 | ABAC ポリシー設計の入力 |

---

## 3. 採用案: 案A 最小構成

### 3.1 構成図

```
[User / Service Account]
        │
        ▼
[Azure API Management(既存共有)]
   - JWT 検証 (Entra ID)
   - スロットリング / Subscription 管理
   - 組込みキャッシュ(短期TTL, 共通レスポンス向け)
        │
        ▼
[Azure App Service (Linux)]
   - OpenAPI 3.1 準拠 REST API
   - プロセス内 LRU キャッシュ (属性, マスタ, スキーマ)
   - ABAC 認可ロジック (PyCasbin 埋め込み PDP)
   - BigQuery クライアント (接続プール)
        │
        ├─→ [Open-GIM 属性ストア] (ExpressRoute) ← ユーザー属性
        │
        └─→ [BigQuery] (Proxy 経由) ← 通常ビュー / 一部マテビュー
```

### 3.2 メリット

- **追加 Azure サービスゼロでスタート可能**。新規サービスの社内承認・コスト・運用学習コストを最小化
- **責任境界が単純**: 全ロジックが App Service に集約され、トラブルシューティングと監査が容易
- **App Service は枯れた技術**: IaC（Bicep/Terraform）、デプロイスロット、自動スケール、カスタムドメイン、診断設定が成熟
- **行・列レベル制御を一箇所で管理**: 中央チームのガバナンス責務範囲が明確
- **第2段階以降への漸進的拡張が容易**: 必要に応じて BI Engine + マテビュー（案B 要素）や Azure SQL ホットストア（案C 要素）を個別追加できる
- **L8（iPaaS 境界）が引きやすい**: APIM = 入口統制、App Service = ビジネスロジック、将来の iPaaS = 非同期統合の三層が明確

### 3.3 デメリット / 既知リスク

| # | リスク | 緩和策 |
|---|--------|--------|
| R1 | **キャッシュミス時のレイテンシが BigQuery 直撃** | (a) 頻出クエリはマテビュー化、(b) BigQuery 側パーティション/クラスタリング最適化、(c) ミス時 SLO を別途定義（例: 95%tile 3 秒以内） |
| R2 | **APIM 組込みキャッシュは RLS 適用済レスポンスでヒット率が低い** | キャッシュは「共通参照系（マスタ・スキーマ・公開メタデータ）」中心に限定。RLS 適用エンドポイントはキャッシュ対象外と割り切る |
| R3 | **APIM 組込みキャッシュは揮発性・APIM アップデート時クリア** | 長期キャッシュ前提の設計をしない。TTL は短く（秒〜分） |
| R4 | **プロセス内キャッシュはインスタンス間で共有されない** | スケールアウト想定エンドポイントでは、共有が必要なデータをキャッシュしない。属性キャッシュは TTL を短く（30〜60 秒） |
| R5 | **Proxy 障害時に API も停止**（ホットパスが BigQuery 依存） | Proxy 冗長性確認。BigQuery クライアントのリトライ/タイムアウト設計を厳密化 |
| R6 | **属性参照のため Open-GIM へ毎リクエスト問い合わせが発生** | プロセス内 LRU で属性をキャッシュ（TTL 短め）。Open-GIM 側に Read レプリカがあれば活用 |

### 3.4 設計上の確定事項

| 領域 | 方針 |
|---|---|
| L1 複雑ロジック配置 | **BigQuery ビュー（基礎）+ 必要に応じてマテビュー（頻出統合モデル）+ API 層（ユーザー文脈依存の組み立て）** |
| L2 キャッシュ配置 | **APIM 組込み + プロセス内 LRU のみ**。Redis 系は将来 C2 制約が緩和されるまで採用しない |
| L3 行・列レベル制御 | **App Service 内 ABAC**。エンジンは **PyCasbin（埋め込み）で確定**（→ `../04-research/abac-authz-library-comparison.md`、`../03-authorization/pycasbin/`）。外部 PDP（Cerbos/OPA）は将来オプション。BigQuery RLS は補助的にも使わない（二重管理回避） |
| L4 ランタイム | **App Service（メイン API）+ Functions（任意で補助バッチ: マテビューウォーマー、Webhook 等）** |
| L5 API 設計規約 | OpenAPI 3.1 準拠、リソース指向 REST、JSON レスポンス。具体規約（JSON:API / OData / 独自）は別途比較 |
| L6 OpenAPI 連携 | スキーマファースト推奨（中央チームのガバナンスと AI 支援開発の双方に有利） |
| L7 SA 属性管理 | SQL Server 同居案を第一候補に検討（人間ユーザーと同じ属性ストアで一元管理） |
| L8 iPaaS 境界 | APIM=入口統制、App Service=ビジネスロジック、iPaaS=非同期統合・既存システム連携 |

---

## 4. 採用見送り案（将来比較用に保持）

### 4.1 案B: BigQuery 側で殴る（BI Engine + マテリアライズドビュー寄せ）

#### 構成概要

```
[User/SA] → APIM(組込みキャッシュ) → App Service
                                     ├─ JWT検証, ABAC
                                     ├─ SQL Server (属性)
                                     └─ BigQuery (Proxy)
                                          ├─ BI Engine 予約 (in-memory)
                                          └─ マテリアライズドビュー
                                            ↑
                              [Functions(Timer)] ← MV ウォーマー
```

#### メリット

- BI Engine の in-memory 加速で、キャッシュミス時でも数百 ms が狙える
- マテビューは事前計算結果を周期的にキャッシュし、繰り返しクエリのレイテンシを大幅低減（ゼロメンテで最新データ）
- App Service 側のコードがシンプル（生 SQL を薄く保てる）
- Azure 側に新規サービス追加なし（案A と同じ Azure リソース構成）

#### デメリット

- BI Engine の予約コスト・サイズ管理が運用負荷として乗る
- 行レベル制御の二重管理リスク（BigQuery RLS と App Service ABAC）
- Proxy 依存度が最大（ホットパスが必ず BigQuery を叩く）→ Proxy 障害＝サービス停止
- ベンダー依存度が GCP 側に偏る
- BigQuery エディションによっては BI Engine / マテビューの利用に制約あり

#### 採用見送り理由

- 案A のリスク R5（Proxy 障害）が最大化する方向の設計
- 行レベル制御の二重管理がガバナンス上の長期負債になる
- 案A の「高速化オプション」として **個別 MV を必要箇所に追加することは可能** であり、案B 全面採用の必要性が乏しい

#### 再検討トリガー

- 定型クエリが多数を占め、マテビュー化のコスパが明確に出るパターンが見えてきたとき
- BigQuery エディションが BI Engine 利用に十分なグレードであることが確認できたとき

---

### 4.2 案C: Azure 側ホットストア（Azure SQL に同期）

#### 構成概要

```
[BigQuery]  ←─ Functions(Timer) ──→ [Azure SQL Database]
                                          ↑
[User/SA] → APIM → App Service ───────────┘
                       ├─ JWT検証, ABAC
                       └─ SQL Server (属性)
```

- Cosmos DB / PostgreSQL は C3 制約により不可。**Azure SQL のみ**が中間ストア候補
- 同期方式は Azure Data Factory または Functions タイマートリガで実装

#### メリット

- **ホットパスのレイテンシが最も読みやすい**（Azure 内クローズドで完結、50〜200ms）
- **L9 Proxy 問題をホットパスから排除**。Proxy 障害時もユーザー API は応答可能（鮮度劣化のみ）
- Azure SQL 標準の RLS / ビューを活用でき、ABAC との二段構えで安全
- 将来のデータレジデンシー要件にも備えられる
- Functions の補助利用が必須となり、「App Service or Functions のみ」制約下で **両方を最大活用できる唯一の案**

#### デメリット

- データ鮮度の劣化（同期間隔次第。数分〜数時間）
- 同期パイプラインという新規ワークストリームが発生（ADF / Functions + dbt 等）
- スキーマ二重管理（BigQuery と Azure SQL の両方）
- 初期構築コストが3案中最大
- Azure SQL は Cosmos DB と比較し、ポイントリードレイテンシが劣る（10〜数十 ms vs 1 桁 ms）

#### 採用見送り理由

- 初期立ち上げコスト・新規ワークストリーム追加が現フェーズに対して重い
- 鮮度要件が現時点で確定しておらず、同期間隔の合意が困難
- 案A から段階的に移行可能なため、最初から採用する必要性が乏しい

#### 再検討トリガー

- 案A 運用中に特定 API が SLO を恒常的に満たせないことが判明したとき
- Proxy 信頼性が想定より低いことが U2 調査または運用で判明したとき
- データレジデンシー要件が新たに発生したとき
- 該当エンドポイントだけ部分的に案C 化する「ハイブリッド」を推奨（全面切替は不要）

---

## 5. 横断比較表

| 観点 | **案A 採用** | 案B BQ寄せ | 案C Azure SQL ホットストア |
|---|---|---|---|
| キャッシュヒット時レイテンシ | ◎ 100-300ms | ◎ 100-300ms | ◎ 50-200ms |
| キャッシュミス時レイテンシ | △ 1-数秒 | ○ 数百ms-1s | ◎ 100-500ms |
| データ鮮度 | ◎ リアルタイム | ◎ MV更新間隔依存 | △ 同期間隔依存 |
| Proxy 依存度（L9 リスク） | 中（ミス時のみ） | **最大**（毎回） | **最小**（バッチのみ） |
| 初期構築コスト | 小 | 小〜中 | 大 |
| 運用負荷 | 小 | 中（BQ最適化） | 大（同期基盤） |
| 行・列制御の実装容易性 | ○ | △（二重管理） | ◎ |
| 追加 Azure サービス | なし | なし | Azure SQL + Functions |
| Functions 活用余地 | 限定的 | 補助 | 必須（同期役） |
| 第2段階以降の拡張性 | ○ | △ | ◎ |
| ベンダーロックイン | 中 | GCP 高 | Azure 高 |
| 現制約下での適合度 | **◎** | ○ | △（重い） |

---

## 6. 採用判断の理由

1. **現フェーズは「API 提供の実現」が直近ミッション**。最小サービス構成で立ち上げ、運用知見を蓄積する案A が最もリスクが低い
2. **追加 Azure サービスを使わない構成は社内承認パスが最短**。C2/C3 で見られる承認制約環境では、これ自体が大きな価値
3. **案A は将来の案B/案C 要素を部分採用できる出発点**。マテビュー追加（案B 要素）も Azure SQL ホットストア追加（案C 要素）も後付け可能
4. **L9 Proxy 依存リスクはあるが、Proxy はどの案でも必要**（案C のバッチ側にも必要）。案A 固有の問題ではない
5. **「Redis 不可」「Azure サービス追加に承認必要」という社内環境は、シンプル構成と相性が良い**

---

## 7. 前提が変わったときの再検討トリガー

| トリガー | 候補となる選択肢 |
|---|---|
| Redis 系（Azure Managed Redis 等）が利用可能になった | 案A + Redis 外部キャッシュ追加。RLS 適用済レスポンスのキャッシュも可能になり、レイテンシ改善大 |
| Container Apps / AKS が利用可能になった | バックエンドランタイム再評価。OPA サイドカー構成や、より柔軟なスケール戦略が可能に |
| Cosmos DB が利用可能になった | 案C 系の選択肢拡大。Azure SQL より低レイテンシのホットストア構築が可能に |
| Proxy 仕様が NG または不安定と判明（U2） | 案C への部分移行を緊急検討。ホットパスを Azure 側に逃がす必要 |
| データレジデンシー要件が発生 | 案C への段階移行 |
| 提供モデルが「定型 SQL × 多数」に集約 | 案A + マテビュー部分採用（案B 要素のハイブリッド化） |
| 特定 API が SLO を恒常的に満たせない | 該当 API のみ案C 化（Azure SQL ホットストア部分採用） |
| BigQuery エディションが BI Engine 対応に十分 | 案A + BI Engine 部分採用 |

---

## 8. 次のアクション

### 8.1 設計フェーズ着手前の必須調査

| # | 項目 | 重要度 | 不明事項 ID |
|---|---|---|---|
| 1 | **Proxy 仕様確認**（認証方式・対応プロトコル・BigQuery クライアント疎通実機検証） | **最高** | U2 |
| 2 | **App Service プラン制約の確認**（Basic / Standard / Premium、Linux/Windows） | 高 | U1 |
| 3 | **既存 APIM の Subscription / Product 運用ルール確認** | 高 | U3 |
| 4 | **Open-GIM（認可属性ストア）スキーマ調査**（実体・到達経路がオンプレ SQL Server と同一かの確認含む） | 高 | U4 |
| 5 | BigQuery エディション・予約状況確認 | 中 | - |
| 6 | Entra ID テナント設定・SA 運用方針確認 | 中 | - |

### 8.2 設計フェーズの判断項目（案A 内部の論点）

| # | 論点 | 内容 |
|---|---|---|
| D1 | 言語・FW 選定 | **【確定】Python / FastAPI を採用**（→ `runtime-framework-decision.md`）。AI 支援開発相性・エコシステム成熟度・App Service ネイティブ対応（2026 年強化）・BigQuery/Entra/認可ライブラリ成熟が決め手。Litestar / .NET 9 / Go・Huma は再検討トリガー付きで見送り |
| D2 | ABAC 実装方式 | **【確定】PyCasbin（埋め込み PDP）を採用**（→ `../04-research/abac-authz-library-comparison.md`、`../03-authorization/pycasbin/`）。外部 PDP（Cerbos/OPA）は将来オプション、`row_filter` 生成・列マスク・BigQuery 翻訳層は自前で補完 |
| D3 | API 設計規約（L5） | JSON:API / OData / 独自 のページング・フィルタ表現比較 |
| D4 | OpenAPI 連携方式（L6） | スキーマファースト vs コードファースト |
| D5 | バージョニング戦略 | URI path（/v1/...）vs Header（Accept-Version） |
| D6 | プロセス内キャッシュライブラリ | 言語依存（.NET: IMemoryCache / Python: cachetools / Node: lru-cache） |
| D7 | 監視・トレーシング | Application Insights / Log Analytics / Datadog の役割分担 |
| D8 | デプロイ戦略 | デプロイスロット運用、Blue/Green、カナリア |

### 8.3 PoC スコープ案

最小 PoC で検証すべき要素:

1. APIM → App Service → BigQuery（Proxy 経由）の疎通とレイテンシ計測
2. Entra ID JWT 検証 → SQL Server 属性参照 → BigQuery クエリの一連フロー
3. ABAC による行レベル WHERE 注入の動作
4. APIM 組込みキャッシュ + プロセス内 LRU のキャッシュヒット率測定
5. キャッシュミス時のレイテンシ実測（→ SLO 妥当性確認）

---

## 9. 改訂履歴

| 日付 | 版 | 内容 |
|---|---|---|
| 2026-05-26 | 1.0 | 案A 採用決定、案B/案C を将来比較用に保持 |
| 2026-06-03 | 1.1 | 言語・FW を Python / FastAPI に確定（D1 クローズ、`runtime-framework-decision.md` 参照） |
| 2026-06-04 | 1.2 | L3/D2 の認可エンジンを **PyCasbin（埋め込み）に確定**（外部 PDP は将来オプション）。認可属性ストアの正本呼称を **Open-GIM** に統一 |
