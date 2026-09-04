# 12. GCPリソース命名規則（パイプライン系）

BigQuery以外のGCPリソース名は全て **`kebab-case`（全小文字・ハイフン区切り）** とする（`10` 章 1節）。

## 1. 一覧

| リソース | 命名パターン | 例 |
| :--- | :--- | :--- |
| GCPプロジェクト | `{system}-{env}` | `example-prd` |
| GCSバケット | `{system}-{env}-{purpose}` | `example-prd-raw`（`13` 章） |
| Cloud Functions (Gen2) | `cf-{関数名}` | `cf-gcs-csv-trigger`, `cf-slack-notifier` |
| Cloud Run Jobs | `crj-{job名}` | `crj-user-sync`, `crj-dbt-executor` |
| Cloud Run Service | `crs-{object}` | `crs-metadata-api` |
| Cloud Workflows | `wf-{pipeline}-{freq}` | `wf-main-daily`, `wf-sales-hourly` |
| Cloud Scheduler | `sch-{起動対象名}` | `sch-wf-main-daily` |
| Pub/Sub トピック | `ps-{event}` | `ps-ingest-completed`, `ps-pipeline-failed` |
| Pub/Sub サブスクリプション | `sub-{topic名}-{consumer}` | `sub-ps-pipeline-failed-alert` |
| Eventarc トリガー | `evt-{source}-{action}` | `evt-gcs-raw-created` |
| BigQuery DTS | `dts-{source_system}-{object}` | `dts-salesforce-account` |
| Dataform リポジトリ | `{system}-dataform` | `example-dataform` |
| Dataform Release Config | `rc-{env}` | `rc-prd` |
| Dataform Workflow Config | `wfc-{env}-{freq}` | `wfc-prd-daily` |
| Artifact Registry | `ar-{purpose}` | `ar-images` |
| Secret Manager | `sec-{用途}` | `sec-user-api-key`, `sec-notification-webhook-url` |
| サービスアカウント | `sa-{役割}`（`30` 章） | `sa-ingest`, `sa-transform`, `sa-workflow` |
| ログシンク | `sink-{用途}` | `sink-bq-audit` |
| 通知チャネル | `nc-{手段}-{宛先}` | `nc-email-dataplatform` |
| アラートポリシー | `alert-{対象}-{条件}` | `alert-wf-main-daily-failure` |
| IAM カスタムロール | `{prefix}{役割}`（**lowerCamelCase**・下記の例外） | `dpTransferRunner` |

> **例外: IAMカスタムロール**
> カスタムロールの `role_id` に使えるのは英数字・アンダースコア・ピリオドのみで、**ハイフンが使えない**。
> 「BigQueryの外は `kebab-case`」の原則を適用できないため、**lowerCamelCase** を用いる。
> `{prefix}` はシステム内で統一する（例: `dp` = data platform）。
> 既定ロールが広すぎる場合にだけ作る。まず既定ロールで足りないかを確認すること（`30` 章1節）。

## 2. フォルダ名とリソース名の対応（最重要）

**アプリケーションコードのフォルダ名を、そのままGCPリソース名の一部にする。** 対応が崩れると「どのコードがどのリソースとして動いているか」が追えなくなる。

| フォルダ | リソース名 | 補足 |
| :--- | :--- | :--- |
| `app/run-jobs/{name}/` | `crj-{name}`（`-job` サフィックスは除く） | 1フォルダ = 1 Cloud Run Job |
| `app/functions/{name}/` | `cf-{name}` | 1フォルダ = 1 Cloud Functions |
| `app/workflows/{name}.yaml` | `{name}` | ファイル名＝ワークフロー名（完全一致） |
| `terraform/schemas/{dataset}/{table}.json` | `{dataset}.{table}` | BigQueryのデータセット・テーブルと一致 |

* CI/CDは**このフォルダ単位で `paths` を監視**する（`app/run-jobs/user-sync-job/**` の変更 → `crj-user-sync` のみ再デプロイ）。フォルダとリソースが1対1でないと、変更監視が成立しない。

## 3. Cloud Functions

* 関数名 `cf-{name}`、ソース配置 `app/functions/{name}/`。
* エントリポイント関数名は `main` に統一する。
* ランタイムは `python312`、リージョンは `asia-northeast1`。
* **1関数＝1連携元**。複数ソースを分岐で処理する「万能関数」を作らない。
* Functionsから別のFunctionsを直接呼ぶ連鎖は禁止。順序制御はWorkflowsで表現する。

## 4. Cloud Run Jobs

* ジョブ名 `crj-{name}`、ソース配置 `app/run-jobs/{name}/`。
* Cloud Functionsの上限（メモリ・実行時間9分/60分）に収まらない処理、およびコンテナが必要な処理に使う。
* **イメージタグに `latest` を使わない**。必ずGitのコミットSHAで固定する。
  `{region}-docker.pkg.dev/{gcp_project_id}/ar-images/{name}:{git_sha}`
* 実行時パラメータは環境変数で渡し、ジョブを乱立させない。dbtは1つのジョブ（`crj-dbt-executor`）を `DBT_SELECT` の値だけ変えて使い回す。
* Terraform側は `lifecycle { ignore_changes = [...containers[0].image] }` でイメージを無視する。CI（`run-job-*.yml`）がタグを更新するため、インフラ側の `apply` が巻き戻さないようにするため。

| 環境変数 | 用途 | 例 |
| :--- | :--- | :--- |
| `GCP_PROJECT_ID` | 接続先プロジェクト | `example-prd` |
| `ENV` | 環境コード | `prd` |
| `DBT_SELECT` / `TAGS` | 実行対象の絞り込み | `tag:silver` |
| `ETL_BATCH_ID` | 監査項目に埋める実行ID | `20260904T060000Z-a1b2c3d` |
| `ETL_TS` | 監査項目のタイムスタンプ（UTC, ISO8601） | `2026-09-04T06:00:00Z` |
| `GIT_SHA` | 実行コードのコミットSHA | `a1b2c3d` |
| `TARGET_DATE` | 対象日（バックフィル時に指定） | `2026-03-31` |

## 5. Cloud Workflows / Scheduler

* Workflows名 `wf-{pipeline}-{freq}`。**定義ファイル名をワークフロー名と完全一致**させる（`src/workflows/wf-main-daily.yaml`）。
* 粒度は「取込 → 変換 → 集計 → 通知」を1本で束ねる**パイプライン単位**とする。テーブル単位のワークフローを乱立させない。
* Schedulerジョブ名は `sch-{workflow名}`。**Schedulerは必ずWorkflowsを起動する**（Functions/Jobsを直接叩かない）。
* Schedulerのタイムゾーンは `Asia/Tokyo` に統一し、cron式にJSTで書く。UTC混在を禁止する。
* リトライは Workflows 側で定義する（`22` 章）。Schedulerのリトライは0回にして二重起動を防ぐ。

## 6. BigQuery Data Transfer Service

* 転送設定名 `dts-{source_system}-{object}`。
* 宛先は必ずBRONZEデータセット（`bronze_{source_system}`）。
* **スケジュールはDTS側で持たせず、Workflowsから手動起動（`StartManualTransferRuns`）する**のを既定とする。Workflowsが完了を待って後続を進められるようにするため。
  * 例外: 後続処理が不要な単純同期のみDTS内蔵スケジュールを許容する。
* 書き込み処分（`WRITE_TRUNCATE` / `WRITE_APPEND`）は `20` 章の取込パターンに従い、Terraformで明示する。

## 7. ラベル（全リソース共通・必須）

| キー | 値 | 用途 |
| :--- | :--- | :--- |
| `system` | `example` | システム識別 |
| `env` | `prd` / `stg` / `dev` | 環境別コスト集計 |
| `layer` | `bronze` / `silver` / `gold` / `common` | レイヤー別コスト集計 |
| `owner` | `data-platform-team` | 問い合わせ先 |
| `managed_by` | `terraform` | 手動作成リソースとの区別（ドリフト検知の起点） |

* ラベル値は小文字英数字とハイフンのみ（GCP制約）。日本語不可。
* Terraform側は `default_labels`（providerブロック）で全リソースに一括付与し、リソース個別では `layer` など差分のみ書く。
