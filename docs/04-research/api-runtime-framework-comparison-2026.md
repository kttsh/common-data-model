# API Runtime and Framework Comparison (2026)

> Source: split from previous combined API technology research memo.
> Status: Research memo.
> **決定**: 本比較を踏まえ、言語・FW は **Python / FastAPI を採用**（2026-06）。決定理由は `../02-architecture/runtime-framework-decision.md` を参照。本メモは選定の根拠となる調査記録として保持する。

## TL;DR
- **総合おすすめは「.NET 9 Minimal API」「Python/Litestar(または FastAPI)」「Go/Huma」の3本柱**。.NET は Azure ネイティブ統合と Entra ID / Managed Identity の純正サポートが圧倒的、Litestar は msgspec 由来で Pydantic v2 比 ~12倍速の JSON デコード+バリデーション(jcrist/msgspec 公式ベンチ、CPython 3.11)と OpenAPI 3.1 ネイティブ、Huma は OpenAPI 3.1 ファースト設計と Go 1.22+ 標準 mux 採用で AI 補完との相性が極めて良い。
- **Harness は CI(Cache/Test Intelligence)+ CD(Azure Web App の Slot Deployment / Traffic Shift / Swap Slot ステップ)+ STO + IaCM(Terraform/OpenTofu)+ SCS(SBOM)+ SRM(Azure Log Analytics ヘルスソース)を一気通貫で敷ける**。Bicep は IaCM 非対応のため Terraform に揃えるのが現実解。
- **ABAC は OPA(Rego)+ サイドカー方式が言語非依存で最も将来性がある**。各言語ネイティブの軽量実装(Casbin など)も併用可。BigQuery は公式 SDK が `HTTPS_PROXY` 環境変数を尊重するため Proxy 経由アクセスは問題なし、認証は Workload Identity Federation か Service Account JSON。

---

## Part 1: 言語・フレームワークのフラット比較

評価軸(以下7軸を共通に使う): ①モダンさ・設計思想 / ②開発の活発さ(2025-2026) / ③AI駆動開発との相性 / ④Azure App Service Linux デプロイ容易性 / ⑤BigQuery クライアント成熟度 / ⑥OAuth2/OIDC/JWT 検証 / ⑦ABAC・認可ライブラリ。

## 1.1 Python

### FastAPI
| 観点 | 評価 |
|---|---|
| 設計思想 | Starlette + Pydantic v2、ASGI、型ヒント駆動。**2026 年時点（0.136 系）では OpenAPI 3.1 がデフォルト出力**（旧来の「3.1 準拠は段階的」評価は解消）。ただし既存 APIM は 3.1 を import 互換のみで扱うため、投入時は 3.0 へダウングロードするのが定石。 |
| 活発さ | OpenAI / Anthropic / Microsoft / Netflix / Uber などが実運用、エコシステムは Python 最大級。Tiangolo Inc. が商用バックに。 |
| AI相性 | 学習データが圧倒的。Claude / Copilot / Cursor が最も補完しやすい Python FW。 |
| Azure | App Service Linux の公式 Python ビルドパックでそのまま動く。`gunicorn -w 4 -k uvicorn.workers.UvicornWorker` パターン。 |
| BigQuery | `google-cloud-bigquery` 同期 + `google-cloud-bigquery-storage`(BigQueryReadAsyncClient で gRPC asyncio)。`HTTPS_PROXY` 環境変数を尊重。 |
| JWT/Entra | `msal`(Microsoft 純正)、`PyJWT`、`Authlib` いずれも成熟。サンプルが豊富。 |
| ABAC | OPA REST 呼び出し、または `oso`(Polar)、`casbin` Python。 |

**メリット**: ドキュメントとサンプルの量で他を圧倒、ML / データ系との親和性。
**デメリット**: 大規模化するとルーターの循環 import 問題、Pydantic 起因のパフォーマンス頭打ち、SQLModel の保守の不安(コミュニティ報告)。

### Litestar(旧 Starlite)
| 観点 | 評価 |
|---|---|
| 設計思想 | msgspec 採用で **JSON decode+validate ベンチで Pydantic v2 比 ~12倍**(jcrist/msgspec 公式ベンチ、CPython 3.11、x86 Linux)、HTMX/JWT/キャッシュ/DI/コントローラ階層が**バッテリー内蔵**。OpenAPI 3.1 ネイティブ。スタンドアロンの `@get` デコレータで循環 import 問題なし。 |
| 活発さ | 2025年に大規模 FastAPI 移行事例が増加、Scalar など外部 OSS スポンサーあり(コミュニティドリブン、企業バックは間接的)。 |
| AI相性 | FastAPI ほど学習データはないが、型情報の表現力は Litestar の方が上。LLM 補完は十分実用的だが「FastAPI と勘違いした出力」が混じる頻度はある。 |
| Azure | FastAPI と同じ ASGI なので App Service Linux の Python ビルドパックでそのまま動く。 |
| BigQuery | FastAPI と同じ(言語共通)。 |
| JWT/Entra | 組込みの JWT/Session 認証あり。Entra ID は外部 `msal` 併用。 |
| ABAC | 組込みのガード/`Permissions` 機構あり。OPA と統合容易。 |

**メリット**: モダンさで頭一つ抜けている。リクエスト/レスポンスの DTO 駆動が明確で AI 駆動開発にも向く。
**デメリット**: Stack Overflow の Q&A が FastAPI ほどない、社内エンジニアの学習コストはやや高い。

### その他活発な Python FW
- **BlackSheep**: 高速・型駆動・OpenAPI 自動生成、Microsoft 系統エンジニアに人気。商用バックは個人。
- **Robyn**: Rust + Python ハイブリッドの ASGI。実験的、本番採用には時期尚早。
- **Sanic**: 老舗だが採用は減少傾向。

### Python 総合所見
**Litestar が技術的にはベスト**だが、エコシステム成熟度・採用事例・社内学習コストを考慮すると**FastAPI が現実解**。ただし大規模化が見えているならば最初から Litestar を選ぶ価値あり。

> **決定（2026-06）**: 本基盤は応答時間が BigQuery・SQL Server の I/O 待機に支配され、Litestar の生スループット優位が体感に出にくい。AI 支援開発相性・エコシステム成熟度・App Service ネイティブ対応（2026 年強化）を重視し **FastAPI を採用**。詳細は `../02-architecture/runtime-framework-decision.md`。

---

## 1.2 TypeScript / Node.js

### NestJS
| 観点 | 評価 |
|---|---|
| 設計思想 | Angular ライクの DI/Module/Decorator、Express または Fastify アダプタ選択可。OpenAPI は `@nestjs/swagger` 経由(3.0)。 |
| 活発さ | エンタープライズ採用の事実上の標準。Adidas / Autodesk / Adobe / Roche / Capgemini など。コミュニティ活発。 |
| AI相性 | 学習データ豊富。Decorator パターンは LLM が得意。ただし暗黙の DSL(Module/Provider のセマンティクス)があるため、生成コードの「整合性」を人が確認する必要がある。 |
| Azure | App Service Linux の Node ビルドパックでそのまま動く。Application Insights SDK あり。 |
| BigQuery | `@google-cloud/bigquery`(公式)、`HTTPS_PROXY` 尊重。 |
| JWT/Entra | `@nestjs/passport` + `passport-azure-ad`、または `jose` で JWKS 直接検証。Entra ID サンプル豊富。 |
| ABAC | `nest-casl`、`@casl/ability`、OPA HTTP クライアント。 |

**メリット**: 構造化された大規模開発、新人オンボーディングが速い。
**デメリット**: 起動時間・メモリ使用量が重い、ボイラープレートが多い、Fastify アダプタを使わないと素の Express 相当の性能。

### Fastify
| 観点 | 評価 |
|---|---|
| 設計思想 | JSON Schema ファースト、高速ルーター + 事前コンパイルシリアライザ。**Fastify 公式ベンチマーク(2026年1月1日更新)で Fastify 46,664 req/sec vs Express 9,433 req/sec、約4.95倍**のスループット。プラグインシステム。 |
| 活発さ | OpenJS Foundation、Microsoft / Platformatic / Nearform 採用。コア開発者 10-15名(Matteo Collina、Tomas Della Vedova ら)。 |
| AI相性 | JSON Schema が明示的なので LLM 補完しやすい。 |
| Azure | Node ビルドパックでそのまま。 |
| BigQuery | 同上。 |
| JWT/Entra | `@fastify/jwt`、`fast-jwt`。 |
| ABAC | OPA HTTP、`casbin` Node。 |

**メリット**: 性能と Schema-First の両立、軽量。
**デメリット**: 構造をチーム側で決める必要があり、ジュニアが多いチームでは規約のばらつきが出やすい。

### Hono
| 観点 | 評価 |
|---|---|
| 設計思想 | Web Standards(Fetch API)ベース、Cloudflare Workers/Bun/Deno/Node どこでも同一コードで動く。14KB minify、ゼロ依存。`@hono/zod-openapi` で OpenAPI 3.1 出力可能。 |
| 活発さ | v4.12 系で安定、Cloudflare(D1, KV, Workers Logs)、Clerk、Unkey、OpenStatus、cdnjs などが本番採用。Cloudflare 社内製。 |
| AI相性 | API が単純で型推論が強力、LLM 出力の品質が高い。 |
| Azure | `@hono/node-server` で Node 環境で動作、App Service Linux 可。ただし「エッジ最適化」が主用途。 |
| BigQuery | 公式 Node SDK そのまま。 |
| JWT/Entra | 組込み JWT(HS/RS/PS/ES/EdDSA すべて対応)、JWKS は `jose` 併用。 |
| ABAC | 組込みなし、Casbin / OPA 等を併用。 |

**メリット**: 圧倒的に軽量、コールドスタートが速い、エッジ将来性。
**デメリット**: 「社内 REST API + App Service」というユースケースでは Cloudflare / Bun の利点が出にくい。Node 環境ではプラグイン量で Fastify / Nest に劣る場面あり。

### Elysia(Bun ベース)
| 観点 | 評価 |
|---|---|
| 設計思想 | Bun ネイティブ、TypeBox による End-to-End 型安全、Eden Treaty で RPC クライアント自動生成。 |
| 活発さ | サブベンチで Encore.ts より高速、v1.4 系で安定。商用バックなし、ボランティアコミュニティ。 |
| AI相性 | TypeBox スキーマが LLM に明示的に伝わる。 |
| Azure | **Bun ランタイムは App Service Linux で公式サポートされない**(カスタムスタートアップで動かす必要あり)。コンテナ前提だが今回は Container Apps/AKS 不可なので**事実上選択肢から除外**。 |

**結論**: 技術的には魅力的だが、ランタイム制約で今回の構成では非推奨。

### Encore.ts
| 観点 | 評価 |
|---|---|
| 設計思想 | Rust 製ランタイム。**Encore 公式エンジニアリングブログ(encore.dev/blog/event-loops)**によると「9x faster than Express.js, 3x faster than ElysiaJS & Hono」(150 concurrent workers, 10s, oha load tool)。インフラ as 型(Pub/Sub、DB を型として宣言)、Cloud Native 前提。 |
| 活発さ | Encore 社が商用バック、独自プラットフォーム(Encore Cloud)。OSS。 |
| AI相性 | コード生成は得意だが、独自 DSL を理解させる必要あり。 |
| Azure | **AWS / GCP 統合は組込みあり、Azure は未公式**。Docker export して App Service Linux に置く形は可能だが、Encore の旨味(自動プロビジョニング)が消える。 |

**結論**: 今回の Azure 縛り構成では Encore.ts の利点が活かせず非推奨。

### TypeScript 総合所見
**第一推奨は NestJS(Fastify アダプタ)**。エンタープライズ REST API + 大人数チーム + Entra ID + ABAC の組み合わせで最も枯れている。**第二推奨は Fastify 単体**(小〜中規模・性能重視)。Hono は App Service ではなくエッジに振る判断のときに採用すべきで、今回の構成では「使えるが旨味は出ない」。

---

## 1.3 Go

### Huma(OpenAPI ファースト)
| 観点 | 評価 |
|---|---|
| 設計思想 | OpenAPI 3.1 + JSON Schema ネイティブ生成、Go 1.22+ 標準 `http.ServeMux` または chi/gin/fiber アダプタ。リクエスト/レスポンス構造体駆動で**設計が強制される**。最新版は Go 1.25+ 必須。 |
| 活発さ | Daniel G. Taylor(Restish 作者、Telepathy Labs スタッフエンジニア)主導、本番採用増加中(ライブストリーミング動画系で実績、自社サイト記載)。商用バックなし、個人 OSS。 |
| AI相性 | 構造体駆動 + 型情報豊富、Go の static typing で LLM が極めて正確に書ける。「FastAPI-Go 版」として LLM が認識する。 |
| Azure | App Service Linux の Go カスタムランタイム(または single binary を ZIP デプロイ)で動作。 |
| BigQuery | `cloud.google.com/go/bigquery` 公式、context ベースの非同期。`HTTPS_PROXY` 尊重。 |
| JWT/Entra | `golang-jwt/jwt`、`MicrosoftDocs/microsoft-authentication-library-for-go`、`coreos/go-oidc`。 |
| ABAC | OPA Go SDK(`github.com/open-policy-agent/opa/sdk`)、`casbin/casbin`、`cerbos/cerbos`。 |

**メリット**: OpenAPI 3.1 ネイティブが2026年現在ベスト、AI コーディング相性が極めて良い、構造体スキーマからドキュメント・モック・SDK・CLI が自動。
**デメリット**: コミュニティ規模が小さい、商用サポートなし、独自ツールチェイン(Restish 連携など)はオプション。

### Gin / Echo / Fiber / Chi
| FW | 立ち位置 | 特徴 |
|---|---|---|
| Gin | 80k+★、最大シェア(Go 開発者調査で約48%)、最も枯れている | プラグイン豊富、HTTP/2 OK(net/http) |
| Echo | 30k 弱★、エンタープライズ寄り | 機能リッチ、構造化エラーハンドリング |
| Fiber | Fasthttp ベースで最速(simple JSON で約130k req/sec) | **HTTP/2 非対応、stdlib middleware 非互換**(Azure API Management の HTTP/2 終端を使う場合に注意) |
| Chi | 軽量・標準 net/http 互換 | マイクロサービス向け、ミニマル |

**OpenAPI 3.1 観点では Huma が圧倒**、純粋ルーターとして Gin / Chi が無難。Fiber は App Service Linux + APIM の組合せでは HTTP/2 非対応がリスク。

### Go 総合所見
**Huma を第一推奨**。Gin / Chi をベースに Huma を被せる構成が、OpenAPI スキーマファースト × AI 駆動開発 × App Service デプロイすべてのバランスが最も良い。

---

## 1.4 C# / .NET

### .NET 9 Minimal API
| 観点 | 評価 |
|---|---|
| 設計思想 | Microsoft 公式 .NET 9 発表ブログ(devblogs.microsoft.com/dotnet/announcing-dotnet-9/)では「higher throughput and a dramatic drop in memory usage」「The memory drop is due to the Server GC changes」と表現。AOT フレンドリー、`Microsoft.AspNetCore.OpenApi` で OpenAPI 3.0/3.1 ネイティブ生成。 |
| 活発さ | Microsoft の最重要投資、毎年メジャーリリース。コミュニティ最大級。 |
| AI相性 | Copilot は .NET が最も得意、`TypedResults` の型表現が LLM に伝わりやすい。 |
| Azure | **App Service Linux 公式 .NET ビルドパック、デプロイスロット完全サポート、Managed Identity サポートが全 SDK で純正、Application Insights は自動計測**。総合スコアが最高。 |
| BigQuery | `Google.Cloud.BigQuery.V2`(公式)、HttpClient 経由なので Proxy 設定可。 |
| JWT/Entra | **`Microsoft.Identity.Web` が事実上の標準**、Entra ID 統合は最もスムーズ。JWKS キャッシュ自動、`AddMicrosoftIdentityWebApi` 一行で完結。 |
| ABAC | `Microsoft.AspNetCore.Authorization`(Policy / Requirement)+ `Styra/opa-csharp`、`Casbin.NET`。 |

**メリット**: Azure とのネイティブ統合が圧倒的。Managed Identity → Key Vault → SQL Server → Application Insights → Log Analytics がすべて純正 SDK で繋がる。
**デメリット**: 起動時間は Native AOT を使わない限り重め(JIT)。コンテナ運用前提なら AOT 化を視野に。

### ASP.NET Core(フル MVC)
- Minimal API が大半のケースで上位互換になったため、新規 API では Minimal を選ぶのが2026年の常識。
- 大規模で複雑なフィルタ/モデルバインダ/カスタムバリデーション資産がある場合のみ MVC コントローラを継続。

### FastEndpoints(REPR パターン)
| 観点 | 評価 |
|---|---|
| 設計思想 | Request-Endpoint-Response パターンを強制、Endpoint 1クラス = 1エンドポイント。Minimal API ベースで性能はほぼ同等、MVC より速い。 |
| 活発さ | サードパーティ OSS(Microsoft 製ではない)、活発だが商用バックは個人/小規模。 |
| AI相性 | パターンが明示的で LLM 生成しやすい、Vertical Slice Architecture と相性◎。 |
| OpenAPI | FastEndpoints.Swagger で自動生成、ただし Swashbuckle 系の出力と微妙に異なる点に注意。 |
| リスク | サードパーティ依存・ライセンス変更リスク(一部機能商用化議論あり)。 |

### .NET 総合所見
**第一推奨は .NET 9 Minimal API**(あるいは .NET 10 が GA 済みなら直接 .NET 10)。FastEndpoints はチーム規模が大きく「構造を強制したい」場面のみ。

---

## 1.5 Java

### Spring Boot
- 枯れた事実上の標準、Entra ID 連携(`spring-cloud-azure-starter-active-directory`)、Application Insights Agent あり。
- App Service Linux の Java ビルドパックで `jar` をそのままデプロイ可。
- 起動時間(JVM 3-6秒)・メモリ(150MB+)が重い。
- AI 補完の学習データは Java 最大。

### Quarkus
- GraalVM Native Image で起動 ~50ms、Java Code Geeks 2026 系ベンチで Native 時 RSS ~70 MB(Spring Boot Native は ~149 MB)、コンテナ密度に効く。
- Red Hat 商用バック、Kubernetes / OpenShift 前提の設計。
- App Service Linux でも Native Image を Docker で動かせばよい(が今回はコンテナ縛り外なので native build → JAR 配置の方が素直)。
- AI 補完は Spring に劣る。

### Micronaut
- コンパイル時 DI、リフレクション排除、Native 約50ms 起動。
- Object Computing 商用バック、サーバーレス向け。
- ライブリロードが弱い、ドキュメントは Quarkus の方が充実。

### Java 総合所見
今回の社内 REST API 用途で「枯れた選択」を取るなら**Spring Boot 3 + Kotlin**(後述)。コンテナ密度・コールドスタートが KPI なら Quarkus。Micronaut は Spring 経験者にはやや異質。

---

## 1.6 Kotlin

### Ktor(JetBrains)
- 軽量・明示的、coroutine ネイティブ、プラグイン式。Spring Boot 比でコールドスタート 2-3秒短縮、メモリ約 1/3(独立ベンチ)。
- OpenAPI は `ktor-openapi` プラグイン or 手動。Spring ほどリッチではない。
- JetBrains 商用バック、Kotlin Multiplatform で iOS / Android 共有可。
- AI 補完は Spring Boot に明確に劣る(学習データ量)。

### Spring Boot(Kotlin)
- 上記 Spring Boot に Kotlin null-safety + coroutines を追加。事実上の Kotlin バックエンドの標準。

### Kotlin 総合所見
**チームに Spring 経験者が多いなら Spring Boot(Kotlin)が素直**。クラウドネイティブ性能を狙うなら Ktor。社内 API で Kotlin を採用する強い理由がない限り、.NET / Python / Go を優先した方がスタック全体が薄くなる。

---

## 1.7 Rust

### Axum
- Tower エコシステム、hyper ベース、型安全。2025年に開発者調査で Actix を抜き Rust 第一位の DX 評価。v0.8 系(2026年1月)。
- OpenAPI は `aide` / `utoipa` 経由(ネイティブではない)。
- Tokio コミュニティ主導、Cloudflare/AWS 等のスポンサー(間接)。

### Actix-web
- 史上最速級、TechEmpower 上位常連。v4.12 系(2025年11月)。
- 学習曲線は Axum より急、Actor モデルの理解が必要。

### Poem
- ミニマル設計、OpenAPI 3 サポート(poem-openapi)、HTTP/3 実験対応。
- エコシステム規模は Axum / Actix に劣る。

### Rust 総合所見
社内 REST API のような I/O バウンドワークロードで Rust を採用する積極理由は乏しい(性能優位がデータベース待機に消える)。**メモリフットプリント・コスト最適化が KPI で、開発者 Rust 経験ありなら Axum**。それ以外なら .NET / Go の方が安全。

---

## Part 1 総合おすすめ(2-3候補)

| 順位 | 言語/FW | 推奨理由 |
|---|---|---|
| **🥇 1位** | **.NET 9 Minimal API** | Azure ネイティブ統合・Entra ID/Managed Identity 純正・Application Insights 自動計測・OpenAPI ネイティブ。社内 REST API 提供基盤として最もリスクが低い。 |
| **🥈 2位** | **Python / Litestar(または FastAPI)** | データ・AI 系周辺との親和性が高く、BigQuery クライアントが最も成熟。LLM / AI 駆動開発との相性は最高クラス。 |
| **🥉 3位** | **Go / Huma** | OpenAPI 3.1 ファースト設計、シングルバイナリでデプロイが軽い、構造体駆動で AI 補完精度が高い。 |

**避けたほうがよいもの(今回の構成では)**: Bun ベース Elysia(App Service 非公式)、Encore.ts(Azure 未公式)、Fiber(HTTP/2 非対応)。

---
