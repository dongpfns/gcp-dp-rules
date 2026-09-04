# 40. インフラ（Terraform）規約

## 1. 原則

1. **すべてのGCPリソースをTerraformで管理する**。コンソール・`gcloud` での恒久的な変更を禁止する（調査目的の参照は可）。
2. **`.tf` 本体は全環境共通**、環境差分は `environments/{env}.tfvars` の変数値のみで表現する。環境ごとに `.tf` を分岐させない。
3. **stateは環境ごとに分離**する（バケットは同じ、prefixで分ける）。
4. 手動変更されたリソースは `managed_by = terraform` ラベルの有無で検知する（`12` 章 6節）。

## 2. ディレクトリ構成

```text
terraform/
├── main.tf                プロバイダ / backend / API有効化
├── variables.tf           変数の宣言・型・validation
├── outputs.tf             他から参照する値
├── locals.tf              命名の組み立て（例: "${var.system}-${var.env}-raw"）
├── environments/
│   ├── dev.tfvars
│   ├── dev.secret.tfvars  （.gitignore対象）
│   ├── stg.tfvars
│   └── prd.tfvars
├── iam.tf                 サービスアカウントとロール付与
├── storage.tf             GCSバケット
├── secret_manager.tf      シークレットの入れ物（値は入れない）
├── bigquery.tf            データセット定義 + modules経由のテーブル生成
├── bigquery_dts.tf        Data Transfer Service
├── functions.tf           Cloud Functions（取込）
├── workflows.tf           Cloud Workflows
├── scheduler.tf           Cloud Scheduler
├── pubsub.tf              Pub/Sub
├── transform_dataform.tf  変換基盤A（count で切替）
├── transform_dbt.tf       変換基盤B（Artifact Registry + Cloud Run Jobs、count で切替）
├── logging.tf             ログシンク
├── monitoring.tf          アラートポリシー・通知チャネル
├── modules/               「作り方」（HCL）
│   ├── bigquery_dataset/
│   └── bigquery_table/
└── schemas/               「中身」（列定義JSON）
    ├── silver_common/d_account.json
    └── silver_sales/f_30m_sales_transaction.json
```

**`modules/` と `schemas/` の役割分担**

* `modules/` = 全テーブル共通の作成手順（パーティション・クラスタ・ラベル・削除保護の当て方）。変更すると全テーブルに波及する。
* `schemas/` = テーブルごとの列定義（JSON）。列の追加はここだけを触る。
* この分離により、**列追加のPRがインフラのHCLに触れない**ため、レビューが軽くなる。

## 3. 命名規約

* リソースの Terraform 識別子は、**実リソース名（BigQueryの場合はテーブル名）とスネークケースで完全一致**させる。
  * `resource "google_bigquery_table" "d_account"` — `t_account` / `account_dim` などの別名は禁止。
  * データセットは `resource "google_bigquery_dataset" "silver_common"`。
* 変数名・ローカル値は `snake_case`（`10` 章）。
* リソース名の組み立ては `locals.tf` に集約し、各 `.tf` で文字列を組み立てない。

```hcl
# locals.tf
locals {
  prefix        = "${var.system}-${var.env}"          # example-prd
  bucket_raw = "${local.prefix}-raw"
  labels = {
    system     = var.system
    env        = var.env
    owner      = var.owner
    managed_by = "terraform"
  }
}
```

## 4. 変数（variables.tf）

1. **すべての変数に `type` と `description` を書く**。
2. 環境依存の値に `default` を置かない（`.tfvars` で必ず明示させる）。共通の定数のみ `default` 可。
3. 取りうる値が決まっている変数は `validation` で制限する。

```hcl
variable "env" {
  type        = string
  description = "環境コード（dev / stg / prd）"
  validation {
    condition     = contains(["dev", "stg", "prd"], var.env)
    error_message = "env は dev / stg / prd のいずれか。"
  }
}

variable "partition_granularity" {
  type        = string
  description = "パーティション粒度（DAY / MONTH / YEAR）。10年以上保持するテーブルは MONTH。"
  default     = "MONTH"
  validation {
    condition     = contains(["DAY", "MONTH", "YEAR"], var.partition_granularity)
    error_message = "partition_granularity は DAY / MONTH / YEAR のいずれか。"
  }
}
```

## 5. BigQueryテーブルの定義

* パーティション粒度は `time_partitioning.type` で指定し、**モジュールの変数として外に出す**。`DAY` 固定にしない（`11` 章 3.1）。
* `MONTH` を指定するとDDLは `PARTITION BY DATE_TRUNC(<field>, MONTH)` になる。**クエリ側は従来どおり `<field>` に条件を書けばプルーニングが効く**ため、SQLの修正は不要。
* **粒度の変更は `terraform apply` では反映できない**（`replace` が必要）。初期設計で決める。
* 本番のSILVER/GOLDテーブルは `deletion_protection = true`。
* `require_partition_filter = true` を既定。
* GCSバケットの `lifecycle_rule` は **`Delete` ではなく `SetStorageClass`** を既定とする（`13` 章）。

## 6. state管理

```hcl
terraform {
  backend "gcs" {
    bucket = "example-prd-tfstate"   # -backend-config で環境ごとに切替
    prefix = "platform"
  }
}
```

1. stateバケットは**バージョニング必須**、`force_destroy = false`。
2. stateバケットは Terraform 管理外（ブートストラップスクリプトで作成）とし、鶏卵問題を避ける。
3. **stateにシークレットの値を入れない**（`30` 章 5節）。
4. `terraform apply` はCI/CD（GitHub Actions）からのみ実行し、ローカルからの本番applyを禁止する（`50` 章）。

## 7. モジュール規約

1. 自作モジュールは `modules/` 配下に置き、外部レジストリからのモジュール取得は原則しない（バージョン管理と監査が難しくなる）。
2. モジュールは**単一責務**（`bigquery_dataset` / `bigquery_table` のように1リソース種別）。
3. モジュールの入出力は `variables.tf` / `outputs.tf` に `description` 付きで定義する。
4. モジュールを変更するPRは、**影響を受けるリソースの `terraform plan` 差分をPR本文に貼る**。

## 8. プロバイダ・バージョン

```hcl
terraform {
  required_version = "~> 1.9"
  required_providers {
    google = { source = "hashicorp/google", version = "~> 6.0" }
  }
}

provider "google" {
  project = var.gcp_project_id
  region  = var.region          # asia-northeast1
  default_labels = local.labels # 全リソースに一括付与
}
```

* バージョンは `~>` でマイナーまで固定し、`.terraform.lock.hcl` をコミットする。
* プロバイダのアップグレードは単独PRで行い、`plan` の差分を確認してからマージする。

## 9. 有効化するAPI

`main.tf` の `google_project_service` で明示的に有効化する。

| API | 用途 |
| :--- | :--- |
| `cloudresourcemanager.googleapis.com` | プロジェクト・IAM管理の前提 |
| `iam.googleapis.com` / `iamcredentials.googleapis.com` | SA・権限・Workload Identity |
| `storage.googleapis.com` | GCS |
| `secretmanager.googleapis.com` | Secret Manager |
| `bigquery.googleapis.com` | BigQuery |
| `bigquerydatatransfer.googleapis.com` | Data Transfer Service |
| `cloudfunctions.googleapis.com` / `run.googleapis.com` | Cloud Functions Gen2（実行基盤はCloud Run） |
| `eventarc.googleapis.com` | GCS/Pub/Subイベントトリガー |
| `artifactregistry.googleapis.com` / `cloudbuild.googleapis.com` | 関数・コンテナのビルドと保管 |
| `workflows.googleapis.com` / `workflowexecutions.googleapis.com` | Workflows |
| `cloudscheduler.googleapis.com` | Scheduler |
| `pubsub.googleapis.com` | Pub/Sub |
| `dataform.googleapis.com` | 変換基盤A |
| `logging.googleapis.com` / `monitoring.googleapis.com` | ログ・監視 |

* `disable_on_destroy = false` を設定する（`terraform destroy` で他システムのAPIまで止めないため）。

**API有効化の伝播（初回applyが落ちる原因）**

APIの有効化はGCP側の伝播に数十秒かかる。同一の `apply` の中で「有効化」と
「そのAPIを使うリソースの作成」が並ぶと、`depends_on` を書いても伝播が間に合わず、
`API has not been used in project ... before or it is disabled` で**初回applyが必ず落ちる**。
`google_project_service` を書いただけでは足りない。

1. **ブートストラップスクリプトで先に有効化し、実際に有効になったことを確認してから** `terraform apply` を行う。
   `gcloud services enable` は戻ってきても使えるようになったとは限らないため、
   `gcloud services list --enabled` で全件が見えるまでポーリングする。
2. **APIリストは1つのファイルを正とし、スクリプトとTerraformの両方がそれを読む。**
   両方に列挙する二重管理は、必ずどちらかが古くなる。

   ```hcl
   # main.tf : required-apis.txt を単一の正として読む
   locals {
     required_apis = [
       for line in split("\n", file("${path.module}/required-apis.txt")) :
       trimspace(line)
       if trimspace(line) != "" && !startswith(trimspace(line), "#")
     ]
   }
   ```

3. Terraform側の `google_project_service` は、**宣言的な正・ドリフト検知の起点として残す**（削除しない）。
4. まだ使うリソースが存在しないAPIを先回りで有効化しない。
   有効化はそのAPIを使うリソースを追加するPRで、同じPRの中で行う。
