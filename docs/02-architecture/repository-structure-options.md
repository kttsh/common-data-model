# Repository Structure Options

> Source: split from previous combined API technology research memo.
> Status: Design input.
> **決定（2026-06）**: 言語・FW は **Python / FastAPI を採用**（`runtime-framework-decision.md`）。下記 3 案のうち **#2 Python が採用案**。#1 .NET / #3 Go は比較用に保持。実際のリポジトリ構成は `repository-strategy.md` の `api` リポジトリ（FastAPI、ドメイン単位構成）を正とする。
> **注（整合）**: 下記サンプルは選定前の比較用イメージのため、確定方針と一部食い違う。正は次のとおり——CI/CD は **GitHub Actions**（`.harness/pipelines/` は不採用。`../04-research/ci-cd-delivery-research-2026.md` 参照）、認可は **PyCasbin 埋め込み**（サンプルの `abac.py # OPA HTTP クライアント` ではない。`../03-authorization/pycasbin/` 参照）、#2 の `main.py` 例は Litestar 表記だが採用は FastAPI。

# Part 3: ソースコード構造のイメージ

## 3.1 推奨 #1: .NET 9 Minimal API

```
internal-api-dotnet/
├── .harness/
│   └── pipelines/
│       ├── build.yaml             # CI: lint/build/test/SBOM/scan
│       └── deploy.yaml            # CD: slot deploy/traffic/swap
├── infra/
│   └── terraform/                 # App Service Plan / Slots / KV / DCR
│       ├── main.tf
│       ├── app_service.tf
│       └── variables.tf
├── openapi/
│   ├── openapi.yaml               # OpenAPI 3.1 スキーマ(正)
│   └── spectral.yaml              # Lint ルール
├── src/
│   └── Api/
│       ├── Program.cs             # アプリのエントリポイント・DI 設定・ミドルウェア配線
│       ├── appsettings.json
│       ├── appsettings.Production.json
│       ├── Endpoints/             # Minimal API のグループ別ハンドラ
│       │   ├── UserEndpoints.cs
│       │   └── ReportEndpoints.cs
│       ├── Domain/                # エンティティ・値オブジェクト
│       ├── Application/           # ユースケース層
│       │   └── Reports/GetUserReportsHandler.cs
│       ├── Infrastructure/
│       │   ├── BigQuery/BigQueryClient.cs    # BigQuery クライアント(Proxy/WIF 対応)
│       │   ├── Sql/UserAttributesRepository.cs  # オンプレ SQL Server
│       │   ├── Auth/EntraIdJwtBearer.cs       # JWT 検証ミドルウェア
│       │   ├── Authorization/AbacPolicy.cs    # ABAC ポリシー
│       │   ├── Caching/LruCache.cs            # プロセス内 LRU
│       │   └── Logging/AuditLog.cs            # Log Analytics 監査ログ
│       └── Common/
│           └── Configuration/KeyVaultRefs.cs  # Key Vault 参照
└── tests/
    ├── Api.UnitTests/
    ├── Api.IntegrationTests/
    └── Api.ContractTests/          # Pact / Schemathesis
```

### 主要ファイル役割

**`Program.cs`**(エントリポイント。DI と middleware を1ファイルで配線):
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddAzureKeyVault(new Uri(builder.Configuration["KeyVault:Uri"]!),
                                       new DefaultAzureCredential());

// Entra ID 認証(JWKS キャッシュは MS 純正で自動)
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("EntraId"));

// ABAC
builder.Services.AddAuthorization(o =>
{
    o.AddPolicy("CanReadReport", p => p.RequireAssertion(ctx =>
        AbacPolicy.Evaluate(ctx, action: "read", resource: "report")));
});

// BigQuery(Proxy 対応)
builder.Services.AddSingleton<IBigQueryGateway, BigQueryGateway>();
// オンプレ SQL Server
builder.Services.AddSingleton<IUserAttributesRepository, UserAttributesRepository>();
// プロセス内 LRU
builder.Services.AddSingleton(typeof(LruCache<,>));

builder.Services.AddOpenApi();   // OpenAPI 3.1 自動生成
builder.Services.AddApplicationInsightsTelemetry();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapOpenApi();
app.MapHealthChecks("/healthz");
app.MapUserEndpoints();
app.MapReportEndpoints();
app.Run();
```

**`Infrastructure/BigQuery/BigQueryClient.cs`**(BigQuery クライアント擬似コード):
```csharp
public class BigQueryGateway : IBigQueryGateway
{
    private readonly BigQueryClient _client;
    public BigQueryGateway(IConfiguration cfg)
    {
        // HTTPS_PROXY は HttpClientFactory 経由で適用、SDK は環境変数を尊重
        Environment.SetEnvironmentVariable("HTTPS_PROXY", cfg["BigQuery:Proxy"]);

        // Workload Identity Federation または SA JSON(Key Vault から読み出し)
        var credPath = cfg["BigQuery:CredentialPath"];
        _client = BigQueryClient.Create(cfg["BigQuery:ProjectId"],
            GoogleCredential.FromFile(credPath));
    }
    public async Task<IReadOnlyList<T>> QueryAsync<T>(string sql, CancellationToken ct)
    {
        var job = await _client.CreateQueryJobAsync(sql, parameters: null, cancellationToken: ct);
        var results = await job.GetQueryResultsAsync(cancellationToken: ct);
        return results.Select(MapRow<T>).ToList();
    }
}
```

**`Infrastructure/Auth/EntraIdJwtBearer.cs`**: `Microsoft.Identity.Web` の `AddMicrosoftIdentityWebApi` 呼び出しで完結。JWKS は MSAL が自動キャッシュ、`appsettings.json` の `EntraId:Authority` と `Audience` を設定するだけ。

**`Infrastructure/Authorization/AbacPolicy.cs`**: OPA HTTP クライアント(`Styra/opa-csharp`)経由で Rego ポリシー評価、または軽量自前(`AuthorizationHandlerContext` の Claim と Resource 属性を組合せた述語評価)。

---

## 3.2 推奨 #2: Python / FastAPI 【採用】（Litestar は将来オプション）

```
internal-api-python/
├── .harness/
│   └── pipelines/{build,deploy}.yaml
├── infra/terraform/
├── openapi/
│   ├── openapi.yaml
│   └── spectral.yaml
├── pyproject.toml          # ruff / mypy / pytest 設定
├── src/
│   └── app/
│       ├── main.py                  # Litestar/FastAPI のエントリ
│       ├── settings.py              # pydantic-settings + Key Vault refs
│       ├── api/
│       │   ├── users.py
│       │   └── reports.py           # ルートハンドラ
│       ├── domain/
│       ├── services/
│       │   └── report_service.py
│       ├── infrastructure/
│       │   ├── bigquery_client.py   # BigQuery クライアント(Proxy 対応)
│       │   ├── sqlserver_client.py  # pyodbc / pymssql
│       │   ├── auth_jwt.py          # Entra ID JWT 検証
│       │   ├── abac.py              # OPA HTTP クライアント
│       │   ├── lru_cache.py         # functools.lru_cache + cachetools.TTLCache
│       │   └── audit_log.py         # azure-monitor-opentelemetry-exporter
│       └── __init__.py
└── tests/
    ├── unit/
    ├── integration/
    └── contract/
```

**`main.py`**:
```python
from litestar import Litestar
from litestar.openapi import OpenAPIConfig
from app.api import users, reports
from app.infrastructure.auth_jwt import EntraIdAuth
from app.infrastructure.abac import abac_guard

app = Litestar(
    route_handlers=[users.router, reports.router],
    openapi_config=OpenAPIConfig(title="Internal API", version="1.0.0",
                                  openapi_version="3.1.0"),
    on_app_init=[EntraIdAuth().configure],
    guards=[abac_guard],
)
```

**`infrastructure/bigquery_client.py`**(擬似コード):
```python
import os
from google.cloud import bigquery
from google.oauth2 import service_account

class BigQueryGateway:
    def __init__(self, cfg):
        # SDK は HTTPS_PROXY / HTTP_PROXY を尊重
        os.environ["HTTPS_PROXY"] = cfg.proxy_url
        creds = service_account.Credentials.from_service_account_file(cfg.sa_json_path)
        self._client = bigquery.Client(project=cfg.project_id, credentials=creds)

    async def query(self, sql: str, params: list[bigquery.ScalarQueryParameter]):
        job_config = bigquery.QueryJobConfig(query_parameters=params)
        job = self._client.query(sql, job_config=job_config)
        # 行ストリーム読込みは BigQueryReadAsyncClient(Storage API)推奨
        return [dict(r) for r in job.result()]
```

**`infrastructure/auth_jwt.py`**: `msal` または `PyJWT + httpx` で JWKS を取得・キャッシュ、`aud` / `iss` / `exp` 検証。

---

## 3.3 推奨 #3: Go / Huma

```
internal-api-go/
├── .harness/pipelines/{build,deploy}.yaml
├── infra/terraform/
├── openapi/
│   └── openapi.yaml          # huma が生成、リポジトリの正と diff 比較
├── go.mod
├── cmd/
│   └── server/
│       └── main.go           # エントリポイント、huma API 初期化
├── internal/
│   ├── api/
│   │   ├── users.go
│   │   └── reports.go        # huma.Register でハンドラ登録
│   ├── domain/
│   ├── service/
│   ├── bigquery/
│   │   └── client.go         # cloud.google.com/go/bigquery, Proxy 対応
│   ├── sqlserver/
│   │   └── repo.go           # microsoft/go-mssqldb
│   ├── auth/
│   │   └── jwt.go            # coreos/go-oidc + golang-jwt
│   ├── abac/
│   │   └── opa.go            # open-policy-agent/opa/sdk
│   ├── cache/
│   │   └── lru.go            # hashicorp/golang-lru/v2
│   └── audit/
│       └── log.go            # Azure Monitor OpenTelemetry Exporter
└── test/
    ├── unit/
    ├── integration/
    └── contract/
```

**`cmd/server/main.go`**:
```go
func main() {
    cfg := loadConfig()  // Key Vault refs via DefaultAzureCredential
    router := http.NewServeMux()
    api := humago.New(router, huma.DefaultConfig("Internal API", "1.0.0"))

    bqClient := bigquery.NewClient(cfg)
    sqlRepo  := sqlserver.NewRepo(cfg)
    jwtMW    := auth.NewEntraJWT(cfg)
    opa      := abac.NewOPA(cfg)

    huma.UseMiddleware(api, jwtMW.Middleware, opa.Middleware)
    api.Register(reports.GetReports(bqClient, sqlRepo))
    http.ListenAndServe(":8080", router)
}
```

**`internal/bigquery/client.go`**(擬似コード):
```go
func NewClient(cfg Config) *Client {
    // HTTPS_PROXY を環境変数で適用(Go の net/http は尊重)
    os.Setenv("HTTPS_PROXY", cfg.ProxyURL)
    ctx := context.Background()
    creds := option.WithCredentialsFile(cfg.SAJSONPath)
    bq, _ := bigquery.NewClient(ctx, cfg.ProjectID, creds)
    return &Client{bq: bq}
}
func (c *Client) Query(ctx context.Context, sql string) ([]Row, error) {
    q := c.bq.Query(sql)
    it, err := q.Read(ctx)
    if err != nil { return nil, err }
    var rows []Row
    for { var r Row; err := it.Next(&r)
        if err == iterator.Done { break }
        if err != nil { return nil, err }
        rows = append(rows, r) }
    return rows, nil
}
```

---

## Part 3 総合所見

3言語ともディレクトリ構造は**「①OpenAPI スキーマ正本」「②Infrastructure 層(BigQuery/SQL/Auth/ABAC/Cache/Audit)」「③.harness パイプライン」「④infra/terraform IaC」の四つを必ず分離**するのが原則。BigQuery クライアントは全言語で `HTTPS_PROXY` 環境変数を尊重するため Proxy 経由アクセスはコード変更不要、認証は **Workload Identity Federation(Service Account JSON 不要)**を選べると最も運用が楽になる。Entra ID JWT 検証は .NET なら `Microsoft.Identity.Web`、Python なら `msal` または `PyJWT`、Go なら `coreos/go-oidc` がそれぞれ事実上の標準。

---
