# Authorization Strategy

権限制御の全体方針の入口。確定した方針をここにまとめ、**未決論点は [`open-issues-and-options.md`](open-issues-and-options.md) を正本**とする（本書では蒸し返さない）。

---

## 略語・用語

略語・用語は [`glossary.md`](glossary.md) を参照（認可ドキュメント共通の正本）。

---

## 1. 確定した方針

「モデル単位の権限制御」という方向性は妥当。叩き台レビューを経て、次の方針が固まっている。実体は各正本を参照。

| # | 方針 | 要点 | 正本 |
|---|---|---|---|
| 1 | 認証と認可属性の分離（二段構え） | 認証は Entra ID、認可属性は **Open-GIM**。Entra ID は ID・メール・グループ程度しか持たず、原価センタ・職位・事業単位などは Open-GIM が正本。鮮度は Open-GIM の同期鮮度に依存する | [`open-issues-and-options.md`](open-issues-and-options.md)（論点8,9） |
| 2 | 統制された記法でポリシーを書く | モデルオーナーに自由 SQL（WHERE/SELECT）を書かせない。インジェクション・検証・レビュー・履歴管理のリスクを避け、**PyCasbin の PERM モデル＋ポリシー**と統制された語彙（`row_scope`／列許可）に倒す | [`../90-research/abac-authz-library-comparison.md`](../90-research/abac-authz-library-comparison.md) |
| 3 | 入口開放＋行列制御は常時適用 | 「全社ユーザが申請なしでアクセス」と監査・最小権限を両立させる答えは「デフォルト＝開放だが行・列はモデル別ポリシーで常時適用」。過剰権限付与に読めないよう言い換える | [`open-issues-and-options.md`](open-issues-and-options.md)（論点1） |
| 4 | 職改への追従は条件付き | 属性参照で書けば**人事異動には自動追従**するが、組織改編で原価センタ体系・部署コード体系が再編されると**ルール側の見直しは発生する** | [`open-issues-and-options.md`](open-issues-and-options.md)（論点13,16） |
| 5 | 監査は「適用した条件」を残す | 動的に WHERE を注入するため、実際に適用したフィルタ条件と判定理由を監査ログに残すのがガバナンスの肝 | [`open-issues-and-options.md`](open-issues-and-options.md)（論点14） |
| 6 | 認可は API 層 ABAC・2 段判断 | ①粗い gate（allow/deny、deny は 403 で短絡）→ ②行フィルタ・列マスク生成。`authorize()` は `row_filter`(AST) と `masked_columns` を返す | [`authorization-boundaries-and-interface.md`](authorization-boundaries-and-interface.md) |
| 7 | ランタイム・エンジンの確定 | ランタイムは **ARO（Azure Red Hat OpenShift）上のコンテナで確定**（2026-06 に App Service から切替。補助バッチのランタイムは未決）。言語・FW は **Python / FastAPI**、認可エンジンは **PyCasbin（埋め込み）で確定** | [`../02-architecture/platform-architecture-decision.md`](../02-architecture/platform-architecture-decision.md)、[`../02-architecture/runtime-framework-decision.md`](../02-architecture/runtime-framework-decision.md) |

---

## 2. 未決論点

叩き台レビューで挙がった論点（Open-GIM 参照仕様、統制記法の具体、SA/AI エージェントの扱い、行レベル粒度、属性キャッシュ・レイテンシ ほか）は、対応パターン・メリデメ・推奨・実施内容とともに **[`open-issues-and-options.md`](open-issues-and-options.md) に集約**した。最優先は次の 3 つ。

- 論点3：利用者の BigQuery 直接接続要件の確定（任意 SQL か／接続 ID は誰か）。
- 論点6：条件 AST→BigQuery 翻訳層 / `row_filter` 生成方式。
- 論点8：Open-GIM 参照仕様の確定（同一性・属性項目・突合キー・到達経路）。

---

## 3. 関連ドキュメント

- [`open-issues-and-options.md`](open-issues-and-options.md) — 未決論点と対応パターンの正本。
- [`authorization-boundaries-and-interface.md`](authorization-boundaries-and-interface.md) — PEP/PDP 責務分担と `authorize()` インターフェースの正本。
- [`row-level-filtering-layering.md`](row-level-filtering-layering.md) — 行レベル絞り込みの層分担。
- [`shared-pdp-across-api-and-bigquery.md`](shared-pdp-across-api-and-bigquery.md) — PDP を API/BQ 経路で共用する将来オプション。
- [`pycasbin/`](pycasbin/basic-specification.md) — PyCasbin 基本仕様・ポリシー定義例・BigQuery 展開実装。
