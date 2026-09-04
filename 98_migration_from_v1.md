# 98. v1からの変更点と移行方針

v1（リポジトリ直下の `NAMING_RULE_*.md` 7ファイル）からv2への主な変更点。**v1はアーカイブとして残し、更新しない。**

## 1. ドキュメント構成の変更

| v1 | v2 |
| :--- | :--- |
| `README.md` | `v2/00_INDEX.md` |
| `NAMING_RULE_COMMON.md` | `10_naming_common.md` + `11_naming_bigquery.md` + `21_transform.md`（CDC部分） |
| `NAMING_RULE_DBT.md` | `21_transform.md` 6節 |
| `NAMING_RULE_TERRAFORM.md` | `40_infra_terraform.md` |
| `NAMING_RULE_BQ_NATIVE.md` | `21_transform.md` 4節（`MERGE` の書き方として統合） |
| `NAMING_RULE_ARCH1_DBT_CLOUDRUN.md` | `01_architecture.md` + `12` / `22` / `50` + `gcp-dp-demo/（dbt構成）` |
| `NAMING_RULE_ARCH2_DATAFORM.md` | `01_architecture.md` + `12` / `22` / `50` + `gcp-dp-demo/dataform_project/` |

**方針の変更**: v1は「ツール別・構成案別」に分けていたため、同じ話（監査項目・CDC）が5ファイルに散在していた。v2は**関心事（命名 / 取込 / 変換 / 基盤 / 運用）で分け**、ツール差分は各章の後半に節として置く。

## 2. 命名規則の変更点（要判断）

| 項目 | v1 | v2 | 理由 |
| :--- | :--- | :--- | :--- |
| **データセット名** | `{project_id}_ds_{layer}_{option_name}`<br>例: `prd_ds_silver_common` | `{layer}_{option_name}`<br>例: `silver_common` | 環境はGCPプロジェクトで分離済みで、`prd_` は冗長。`ds_` もデータセットであることが自明。**1プロジェクト複数環境の場合のみ `{env}_` 接頭辞を許容** |
| **GCPリソースの記法** | `wf_main_daily` / `sch_wf_main_daily`（アンダースコア混在） | `wf-main-daily` / `sch-wf-main-daily`（全てハイフン） | 「BigQueryの中は `_`、外は `-`」に統一。GCSがアンダースコア不可、BigQueryがハイフン不可という制約に合わせた |
| **Cloud Run Jobs** | `cr-dbt-runner-{layer}` | `crj-dbt-{layer}` | Jobs（`crj-`）とService（`crs-`）を接頭辞で区別 |
| **プロジェクトID** | `example-{env}` | `{system}-{env}` | 変わらず（プレースホルダ名の明確化のみ） |
| **BRONZE監査項目** | `_bronze_ingested_at` のみ | `_bronze_source_uri` を追加 | 取込元の再現性を確保するため |

### 移行が必要かの判断

* **新規プロジェクト**: v2をそのまま適用する。
* **v1で既に構築済み**: データセットのリネームはコストが高い（下流の全参照を書き換える必要がある）。以下のどちらかを選び、ADRに記録する。
  1. **既存はv1命名のまま維持し、新規追加分からv2命名を使う** — 混在するが移行コストはゼロ。データセット名にレイヤーが含まれる点は両者共通なので実害は小さい。
  2. **一括移行** — `CREATE TABLE ... COPY` で新データセットへ複製 → 下流を切り替え → 旧データセットを一定期間残して削除。ダウンタイムを取れる場合のみ。
* **GCPリソース名（Workflows等）**: リネームは再作成になるが、依存が少ないため比較的安全。次回の変更ついでに寄せる。

## 3. 追加された内容（v1になかったもの）

* `13_naming_storage.md` — GCSバケット構成とオブジェクトパス規約
* `20_ingestion.md` — ELT/ETLの原則、連携元ごとの標準方式、更新方式P1〜P5の選定フロー
* `22_orchestration.md` — Workflows/Scheduler/Functions/Run/DTS の使い分けと実装規約
* `30_iam_security.md` — サービスアカウント設計、アクセス境界、Secret管理
* `50_git_cicd.md` — ブランチ戦略、CI/CDワークフロー一覧、PR規約
* `60_nonfunctional.md` — 可用性・性能・コスト・監視・ログ・品質・DR
* `90_review_checklist.md` — レビューチェックリスト
* `gcp-dp-demo/` — 動かせるテンプレート一式（Terraform + Python + Workflows + dbt / Dataform）

## 4. 変わっていないもの（v1のまま維持）

以下はv1の判断が正しく、v2でもそのまま採用している。

* BRONZE/SILVER/GOLD の3層構成と各層の役割
* BRONZEにテーブル命名を強制しない方針
* `d_` / `f_{grain}_` / `v_` / `t_` の接頭辞
* カラム命名（`_id` / `_cd` / `is_` / `_utc` / `_jst` / 単位サフィックス）
* 監査項目の列名（`etl_loaded_at` 等）
* パーティション粒度の判断基準（4,000パーティション上限）
* CDCの範囲条件3か所ルール
* 派生指標をGOLDに限定する方針
