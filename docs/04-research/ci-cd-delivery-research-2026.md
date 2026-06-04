# CI/CD and Delivery Platform Research (2026)

> Source: split from previous combined API technology research memo.
> Status: **調査資料（採用見送り）**。CI/CD の**実装は GitHub Actions を採用**する（正は `../02-architecture/repository-strategy.md` §6・§10、`../06-data-platform/dataform-operating-model.md`〔不採用〕ではなく `migration-plan.md` の GitHub Actions 前提）。本書の **Harness 一式は不採用**だが、マルチ言語 SAST/SBOM/SRM/Slot Swap を一プラットフォームで束ねたくなった場合の将来比較材料として保持する。下記の `.harness/pipelines/...` 例も Harness 採用時のイメージであり、現行の正規構成ではない。

# Part 2: Harness による CI/CD 敷き詰め（採用見送り・将来比較用）

## 2.0 全体像

```
[GitHub/Harness Code Repo]
        │ push / PR
        ▼
┌─────────────────────────── Harness Pipeline ───────────────────────────┐
│  ① Pre-build (Lint / Static / OpenAPI lint / SAST)                     │
│  ② Build (Dockerfile or Buildpacks → ACR) + SBOM (Syft → CycloneDX)    │
│     + Trivy/Snyk スキャン (STO)                                         │
│  ③ Test (Unit / Integration / Contract / Schemathesis)                 │
│     + Test Intelligence で関連テストのみ実行                            │
│  ④ Deploy to App Service (staging slot → swap)                         │
│  ⑤ Post-deploy (Smoke / App Insights / SRM SLO verify / Rollback)      │
└─────────────────────────────────────────────────────────────────────────┘
        │
        ▼
[Azure App Service Linux] ── Managed Identity ──▶ Key Vault / Log Analytics
        │
        └──▶ SQL Server (ExpressRoute) / BigQuery (Proxy)
```

Harness の関連製品マッピング:
| Harness 製品 | 役割 |
|---|---|
| Harness CI | ①②③(Pre-build / Build / Test) |
| Harness CD | ④(Deploy with Slot Swap) |
| Harness STO | ② (Snyk / Trivy / Semgrep / CodeQL / Aqua / Bandit 等オーケストレーション) |
| Harness SCS | ② (SBOM 生成: Syft または cdxgen、SPDX または CycloneDX) |
| Harness IaCM | Terraform / OpenTofu の State / Drift / Policy 管理(Bicep 非対応) |
| Harness Feature Flags (FME) | リリース後のロールアウト/キルスイッチ |
| Harness CCM | App Service プランのコスト可視化 |
| Harness IDP | サービスカタログ・テンプレート(2.0 系で大幅刷新) |
| Harness SRM | ⑤ SLO/エラーバジェット、Azure Log Analytics ヘルスソース連携 |

---

## 2.1 Pre-build / Lint / Static Analysis フェーズ

### 言語別 Lint(Harness CI のステップで実行)
| 言語 | ツール | Harness CI での扱い |
|---|---|---|
| Python | **ruff**(format + lint 一体)、mypy | `Run` ステップで `ruff check . && mypy src/` |
| Node/TS | **biome**(format + lint)、tsc --noEmit | `Run` ステップで `bun x biome ci . && tsc --noEmit` |
| Go | **golangci-lint**、`go vet` | `Run` ステップで `golangci-lint run ./...` |
| .NET | `dotnet format --verify-no-changes`、Roslyn analyzer | `Run` ステップで `dotnet format --verify-no-changes` |
| Kotlin | **ktlint** または detekt | `Run` ステップで `./gradlew ktlintCheck detekt` |
| Rust | **clippy**(`cargo clippy -- -D warnings`)、rustfmt | 同上 |

### pre-commit との役割分担
- **pre-commit(ローカル + git hook)**: ruff / biome / gofmt など**高速で確定的なもの**だけ。コミット前1秒以内。
- **Harness CI Pre-build ステップ**: pre-commit と**同じコマンド**を再実行(欠落防止)+ mypy / tsc / clippy など**重いもの**+ OpenAPI lint + SAST。
- 「ローカルで通ったが CI で落ちる」を最小化するため、pre-commit と CI は**同じバージョンのツール**を使う(Harness Cache Intelligence でツール自体もキャッシュ)。

### OpenAPI スキーマ lint
- **Spectral**(Stoplight)が業界標準、`@stoplight/spectral-cli`。Harness `Run` ステップで `npx -y @stoplight/spectral-cli lint openapi.yaml`。
- **Vacuum**(Daniel G. Taylor 系列、Go 製・高速)が新興の代替、巨大スキーマで Spectral 比で大幅高速。`vacuum lint openapi.yaml -d`。
- 推奨ルールセット: `spectral:oas` + 社内カスタム(operationId 必須 / セキュリティ定義必須 / x-rate-limit など)。

### SAST(Harness STO 経由)
公式の Supported Scanners ページ(developer.harness.io/docs/security-testing-orchestration/whats-supported/scanners/)が列挙する SAST 対応: Harness SAST(Semgrep ベース)、Aqua Trivy、Bandit、Brakeman、Checkmarx、Checkmarx One、CodeQL、Coverity、Fortify、GitHub Advanced Security、Semgrep、SonarQube、Snyk、Veracode、Wiz、Zap など。

- **Snyk Code**: 公式インテグレーション、JSON または SARIF 取り込み。
- **Semgrep**: Harness STO に Semgrep ステップが組込み済み、SARIF 取り込み。
- **CodeQL**: GitHub Actions と連携、SARIF 結果を Harness STO に Ingest(`Custom Scan` ステップ)。
- **Harness SAST(Semgrep ベース)**: 商用ライセンス不要で組込み利用可。

```yaml
# .harness/pipelines/build.yaml(抜粋)
- step:
    type: Security
    name: Snyk Code (SAST)
    identifier: snyk_sast
    spec:
      config: default
      target:
        type: repository
        name: api-svc
        variant: main
      advanced:
        log:
          level: info
        fail_on_severity: high
      auth:
        access_token: <+secrets.getValue("snyk_token")>
      tool:
        scan_type: source-code
- step:
    type: Run
    name: Spectral OpenAPI Lint
    spec:
      shell: Sh
      command: |
        npx -y @stoplight/spectral-cli lint openapi/openapi.yaml -f junit -o ruleset-report.xml
      reports:
        type: JUnit
        spec:
          paths: [ "ruleset-report.xml" ]
```

---

## 2.2 Build フェーズ

### Docker イメージビルド: Buildpacks vs Dockerfile
| 観点 | Buildpacks (Paketo/Heroku) | Dockerfile |
|---|---|---|
| 設定量 | ほぼゼロ | フル制御 |
| セキュリティ | ベースイメージが自動更新 | 自前で CVE 追跡 |
| カスタマイズ性 | △ | ◎ |
| 推奨 | Python / Node の単純 API | .NET / Go の多段ビルド、社内独自要件あり |

社内 API 提供基盤では**Dockerfile 推奨**(Distroless / chainguard ベース、multi-stage、non-root)。`HTTPS_PROXY` 含む環境変数をビルド時に注入。

### 脆弱性スキャン
- **Trivy**(Aqua、OSS、Harness STO 組込み): イメージ + IaC + secrets を1ツールで。
- **Snyk Open Source**: 依存ライブラリの脆弱性、ライセンス問題。
- 推奨: **Trivy(無償・組込み)を必須、Snyk は商用ライセンスがあれば追加**で重複検出する。

### SBOM 生成
Harness SCS 公式ドキュメント(developer.harness.io/docs/software-supply-chain-assurance/open-source-management/generate-sbom-for-artifacts/)の verbatim:
> "SBOM Tool: Select Syft or cdxgen. For other SBOM tools, go to Ingest SBOM. SBOM Format: Select SPDX or CycloneDX."

- ビルドステージで生成 → Cosign で署名(OIDC keyless 可)→ ACR に `.att` 添付。
- デプロイ前に SBOM ポリシー(OPA)で CRITICAL CVE を含むイメージをブロック(SBOM Policy Enforcement ステップ)。

### Harness Cache Intelligence
- Java / Node / Python / Go の依存キャッシュは**自動**(Gradle / Maven、`node_modules`、`pip`、`go mod`)。
- カスタムパス指定可、TTL は最終更新から 15 日、Harness Cloud ビルドだと管理ストレージは追加コストなし(プラン依存)。
- Docker Layer Caching(DLC)も組込みで利用可、ACR / S3 / GCS / Azure Blob いずれかをバックエンドに。

```yaml
- stage:
    name: Build
    identifier: build
    type: CI
    spec:
      caching:
        enabled: true   # Cache Intelligence ON
      execution:
        steps:
          - step:
              type: BuildAndPushACR
              name: Build & Push to ACR
              spec:
                connectorRef: acr_connector
                registry: myacr.azurecr.io
                subscriptionId: <+secrets.getValue("azure_sub_id")>
                repository: api-svc
                tags: ["<+pipeline.sequenceId>", "latest"]
                dockerfile: Dockerfile
                context: .
                buildArgs:
                  HTTPS_PROXY: <+variable.proxy_url>
          - step:
              type: AquaTrivy
              name: Trivy Image Scan
              spec:
                mode: orchestration
                config: default
                target:
                  type: container
                  name: api-svc
                  variant: <+pipeline.sequenceId>
                advanced:
                  fail_on_severity: critical
                image:
                  type: docker_v2
                  name: myacr.azurecr.io/api-svc
                  tag: <+pipeline.sequenceId>
          - step:
              type: SscaOrchestration   # Harness SCS SBOM 生成
              name: SBOM (Syft + CycloneDX)
              spec:
                tool:
                  type: Syft
                  spec:
                    format: cyclonedx-json
                source:
                  type: image
                  spec:
                    image_name: myacr.azurecr.io/api-svc:<+pipeline.sequenceId>
                attestation:
                  type: cosign
                  spec:
                    private_key: <+secrets.getValue("cosign_key")>
```

---

## 2.3 Test フェーズ

### テスト種別
| 種別 | ツール | Harness での扱い |
|---|---|---|
| 単体 | pytest / jest / go test / xunit / JUnit | `Test` ステップ(Intelligence ON で関連テストのみ実行) |
| 統合 | testcontainers(SQL Server、BigQuery エミュレータ) | `Test` ステップ、コンテナ起動 |
| コントラクト | **Pact**(Consumer-Driven) | Pact Broker 連携、ブローカーに publish |
| API スモーク | Postman/Newman、k6 | `Run` ステップ |
| OpenAPI 適合性 | **Schemathesis**(Property-based、Python)、Dredd | デプロイ後の staging slot に対して `schemathesis run` |

### Harness Test Intelligence(TI)
**Harness CI 公式プロダクトページ(harness.io/products/continuous-integration/test-intelligence)verbatim**:
> "Use AI-powered Test Intelligence to run only Unit Tests related to your code change, slashing test cycle time by up to 80%."

- Java / Python / Ruby / C# / Go を公式サポート。Test グロブパターン(例: `**/*_test.go`)を `enabled: true` + 言語指定で有効化。
- JUnit / TRX / xUnit XML レポートを `paths` で指定。

```yaml
- step:
    type: Test
    name: Intelligent Unit Tests
    spec:
      language: Python
      buildTool: Pytest
      args: -v --cov
      packages: src
      intelligenceMode: true     # Test Intelligence ON
      reports:
        type: JUnit
        spec:
          paths: [ "reports/junit.xml" ]
      parallelism: 4
- step:
    type: Run
    name: Schemathesis Contract Test
    spec:
      shell: Sh
      command: |
        schemathesis run --checks all \
          --base-url https://api-svc-staging.azurewebsites.net \
          openapi/openapi.yaml
```

### OpenAPI と実装の整合性
- **生成スキーマ vs リポジトリのスキーマ**を CI で diff 比較し、ドリフト検知。`openapi-diff` または `oasdiff`(breaking change 検出)を使う。
- スキーマファースト方針なら、リポジトリのスキーマを正 → コード生成(openapi-typescript / NSwag / oapi-codegen)→ コミット差分なしで PR が通る、を CI で強制。

---

## 2.4 Deploy フェーズ(Azure App Service)

### Harness CD の Azure Web App デプロイ
公式ドキュメント(developer.harness.io/docs/continuous-delivery/deploy-srv-diff-platforms/azure/azure-web-apps-tutorial/、Last updated Jan 7, 2026)より、**Azure Web App** デプロイタイプには以下のステップ:

| ステップ名(UI) | YAML type 例 | 役割 |
|---|---|---|
| **Slot Deployment** | `AzureSlotDeployment` | source slot(staging)にイメージ/ZIP をデプロイ |
| **Traffic Shift** | `AzureTrafficShift` | source → target にトラフィック段階的シフト(Canary) |
| **Swap Slot** | `AzureSwapSlot` | staging ↔ production スロットスワップ |
| **Web App Rollback** | `AzureWebAppRollback` | 自動的に rollbackSteps に追加される |

公式の戦略マッピング(verbatim):
> "Basic: Slot Deployment step. No traffic shifting takes place. Canary: Slot Deployment, Traffic Shift, and Swap Slot steps. Traffic is shifted from the source slot to the target slot incrementally. Blue Green: Slot Deployment and Swap Slot steps. All traffic is shifted from the source slot to the target slot at once."

### 必要な Azure RBAC(verbatim)
最低 `Contributor`、加えて `Microsoft.Web/sites/slots/slotsswap/Action`、`Microsoft.Web/sites/slots/Write`、`Microsoft.Web/sites/slots/start/Action`、`Microsoft.Web/sites/slots/stop/Action` 等。

### Linux Web App 制約(verbatim)
> "App Service on Linux isn't supported on Shared pricing tier. You can't mix Windows and Linux apps in the same App Service plan."

### Bicep / Terraform / Pulumi との組合せ
- **Harness IaCM は Terraform と OpenTofu のみ GA**(Pulumi / Crossplane は roadmap、TechTarget で Rohit Reddy Kaliki(principal product manager at Harness)コメント)。
- **Bicep は IaCM 非対応**、Harness ARM 提供ステップも JSON ARM のみで「ARM templates must be in JSON. Bicep isn't supported.」(developer.harness.io/docs/continuous-delivery/cd-infrastructure/azure-arm-provisioning/)。
- 結論: **Terraform に統一**するのが最も摩擦少。Bicep を強く好む場合は `bicep build → ARM JSON` を Harness パイプライン外で生成し、IaCM ではなく `Run` ステップ + `az deployment group create` で適用する妥協案。

### ロールバック戦略
1. **Slot Swap ベース**(推奨): Swap 後に問題発生 → 同じ Swap Slot ステップで戻す(staging に旧版が残っているため)。
2. **Canary Traffic Shift**: Traffic Shift で 10% → 50% → 100% と段階シフト、Harness SRM や App Insights のメトリクスで自動 abort 可。
3. **Web アプリ Rollback ステップ**: `rollbackSteps` に自動追加、`AzureWebAppRollback` が前回成功デプロイの artifact をスロットに再デプロイ。

### GitHub Actions / Azure DevOps Pipelines との比較
| 観点 | Harness | GitHub Actions | Azure Pipelines |
|---|---|---|---|
| Azure ネイティブ | 良(Azure connector) | 良(`azure/webapps-deploy@v3`) | 最良(`AzureWebAppDeploy@1`) |
| パイプライン記述 | YAML + GUI | YAML | YAML(Classic 廃止傾向) |
| Cache 最適化 | Cache Intelligence(自動) | actions/cache(手動) | Cache@2(手動) |
| Test Intelligence | あり | なし(自前) | なし(自前) |
| Slot Swap | 専用ステップ | `az webapp deployment slot swap` | 専用タスク |
| ガバナンス | OPA Policy 統合・RBAC・監査 | Environments / Required reviewers | Approvals + Policy |
| 学習コスト | 中(GUI と YAML 両方) | 低 | 中 |
| 価格 | 商用、デリゲートはセルフホスト | リポジトリ単位 | パイプライン並列単位 |

**判断**: GitHub Actions だけで足りるなら最安だが、**マルチ言語・SAST・SBOM・SLO・FF をワンプラットフォームで縛るなら Harness の旨味が出る**。Azure Pipelines は Azure 専用なら最強だが Harness ほどクロスプラットフォーム連携が広くない。

```yaml
# .harness/pipelines/deploy.yaml(Canary 戦略の抜粋)
stages:
  - stage:
      name: Deploy_Staging
      identifier: deploy_staging
      type: Deployment
      spec:
        deploymentType: AzureWebApp
        service:
          serviceRef: api_svc
        environment:
          environmentRef: prod
          infrastructureDefinitions:
            - identifier: linux_webapp_infra
        execution:
          steps:
            - step:
                name: Slot Deployment
                identifier: slot_deploy
                type: AzureSlotDeployment
                timeout: 20m
                spec:
                  webApp: <+input>
                  deploymentSlot: staging
            - step:
                name: Smoke Test (staging)
                identifier: smoke
                type: Http
                spec:
                  url: https://api-svc-staging.azurewebsites.net/healthz
                  method: GET
                  assertion: <+httpResponseCode> == 200
            - step:
                name: Traffic Shift 20%
                identifier: shift_20
                type: AzureTrafficShift
                timeout: 10m
                spec:
                  traffic: "20"
            - step:
                name: Verify (App Insights / SRM)
                identifier: verify
                type: Verify
                spec:
                  type: Canary
                  monitoredService:
                    type: Default
                  spec:
                    sensitivity: HIGH
                    duration: 10m
                    deploymentTag: <+pipeline.sequenceId>
            - step:
                name: Swap Slot
                identifier: swap
                type: AzureSwapSlot
                timeout: 20m
                spec:
                  targetSlot: production
          rollbackSteps:
            - step:
                name: Web App Rollback
                identifier: rollback
                type: AzureWebAppRollback
                timeout: 20m
                spec: {}
```

> 注: ステップの YAML `type:` 文字列(`AzureSlotDeployment` 等)は公式チュートリアルが UI 名で記述しているため、最終的には `github.com/harness/harness-schema/blob/main/v0/pipeline.json` または Pipeline Studio の Visual→YAML 切替で確定すること。

---

## 2.5 Post-Deploy / 運用

### スモーク・ヘルスチェック
- App Service の **Health check** 機能を有効化(`/healthz` を1分間隔ポーリング、5回失敗でインスタンス置換)。
- Harness `Http` ステップで `/livez` / `/readyz` を Swap 直前に 200 確認、`assertion` で `<+httpResponseCode> == 200`。

### Application Insights / Log Analytics
- App Service Linux に Application Insights Auto-Instrumentation を有効化(.NET / Java / Node / Python サポート)。
- カスタム監査ログは Log Analytics Workspace に **DCR 経由で送信**(または Application Insights → 自動的に Workspace へ)。

### Harness SRM 連携
- SRM の **Azure Log Analytics ヘルスソース**(developer.harness.io/docs/service-reliability-management/monitored-service/health-source/azurelogs/)を作成、KQL で SLI 定義:
  - 例: `requests | where success == false | summarize errorRate = countif(success == false)/count()`
- **エラーバジェット**(公式 verbatim):
  > "Harness SRM generates an error budget automatically based on the SLO target and time window specified. The error budget represents the speed and quality obligations that your service must meet."
  > "Error Budget Remaining = Total Error Budget − Number of Bad Minutes."
- SLO 目標 99.9%(月次)→ 残バジェット 43.2分。Burn Down ダッシュボード提供。
- CD パイプラインの `Verify` ステップで SLO ベース判定 → 自動ロールバック。
- Composite SLO(複数 SLI 合成)も対応、Continuous Verification は Azure Metrics / Azure Log Health いずれも health source として認識。

---

## Part 2 総合所見

**Harness を選んだ前提では、CI(Cache + Test Intelligence)→ Build(Trivy + SBOM via Syft)→ STO(Snyk / Semgrep)→ CD(Azure Web App の Slot 系ステップ)→ SRM(Azure Log Analytics)で完全に閉じる**。Bicep は捨てて Terraform に揃え、IaCM で State 集中管理が最も摩擦が少ない。Feature Flags は商用 SaaS(FME)の値段相応の価値があるが、社内 API でフラグ自体が必要かは要検討(エンタープライズ用途で内製ロールアウト/キルスイッチが欲しいなら採用価値あり)。

---
