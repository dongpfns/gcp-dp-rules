# GCPネイティブ データ基盤 開発規約 v2

GCS / BigQuery / Workflows などGCPネイティブサービスでデータ基盤を構築・運用するための、命名規則・設計方針・開発規約の体系。

## 0. この規約の使い方

| 立場 | 最低限読むもの |
| :--- | :--- |
| はじめて参加する開発者 | `01` → `10` → `11` → `20` → `21` |
| テーブル設計をする人 | `10` `11` `21` `90` |
| パイプラインを作る人 | `20` `22` `12` `13` `90` |
| インフラ・権限を見る人 | `30` `40` `50` `13` |
| レビュー担当 | `90`（チェックリスト）を起点に各章へ |

* 規約に**例外を設ける場合は、必ずリポジトリのADR（`docs/adr/`）に理由を記録**する。口頭合意は規約違反とみなす。
* 章どうしで記述が矛盾した場合は、**番号の小さい章を正**とする（`10` 系の命名原則が最上位）。

## 1. 章立て

### 総論
| # | ファイル | 内容 |
| :--- | :--- | :--- |
| 01 | [`01_architecture.md`](./01_architecture.md) | 全体アーキテクチャ、BRONZE/SILVER/GOLD の役割、構成パターンの選定基準 |

### 命名規則
| # | ファイル | 内容 |
| :--- | :--- | :--- |
| 10 | [`10_naming_common.md`](./10_naming_common.md) | 全リソース共通の命名原則（記法・環境コード・語彙・略語・禁止事項） |
| 11 | [`11_naming_bigquery.md`](./11_naming_bigquery.md) | データセット / テーブル / ビュー / カラム / 監査項目 / 物理設計 |
| 12 | [`12_naming_gcp_resources.md`](./12_naming_gcp_resources.md) | Cloud Run / Functions / Workflows / Scheduler / DTS / Pub/Sub / Secret など |
| 13 | [`13_naming_storage.md`](./13_naming_storage.md) | GCS バケット構成とオブジェクトパス規約 |

### 実装規約
| # | ファイル | 内容 |
| :--- | :--- | :--- |
| 20 | [`20_ingestion.md`](./20_ingestion.md) | ELT/ETL方式、外部連携（Salesforce / Box / 外部API / CSV）、全件・差分更新方式 |
| 21 | [`21_transform.md`](./21_transform.md) | SILVER/GOLD変換規約、監査項目の発行、CDC、dbt / Dataform の実装方式 |
| 22 | [`22_orchestration.md`](./22_orchestration.md) | Workflows / Scheduler / Cloud Run / Functions / DTS の使い分けと実装規約 |

### 基盤・運用
| # | ファイル | 内容 |
| :--- | :--- | :--- |
| 30 | [`30_iam_security.md`](./30_iam_security.md) | サービスアカウント設計、IAM境界、Secret管理、データ保護 |
| 40 | [`40_infra_terraform.md`](./40_infra_terraform.md) | Terraform構成、環境分離、モジュール規約、state管理 |
| 50 | [`50_git_cicd.md`](./50_git_cicd.md) | リポジトリ構成、ブランチ戦略、CI/CD、レビュー規約 |
| 60 | [`60_nonfunctional.md`](./60_nonfunctional.md) | 可用性・性能・コスト・監視・ログ・データ品質・DR・セキュリティ |

### 付録
| # | ファイル | 内容 |
| :--- | :--- | :--- |
| 90 | [`90_review_checklist.md`](./90_review_checklist.md) | レビューチェックリストと「事故りやすい落とし穴」集 |
| 98 | [`98_migration_from_v1.md`](./98_migration_from_v1.md) | v1からの変更点と移行方針 |
| 99 | [`99_glossary.md`](./99_glossary.md) | 用語集・プレースホルダ読み替え表 |

## 2. サンプルプロジェクト

本規約に完全準拠したテンプレート一式を、変換基盤ごとに2つ用意している。

| ディレクトリ | 変換基盤 | リポジトリ構成 |
| :--- | :--- | :--- |
| [`../gcp-dp-demo/`](../gcp-dp-demo/) | **dbt on Cloud Run Jobs**（`enable_dbt` で無効化も可能） | モノレポ1本 |
| [`../gcp-dp-dataform-demo/`](../gcp-dp-dataform-demo/) | **Dataform** | インフラ用 + Dataform専用（切り出し必須） |

取込（Cloud Run Jobs / Cloud Functions）・オーケストレーション（Workflows / Scheduler）・
インフラ（Terraform）・CI/CDの骨格は両者で共通で、**変換層だけが異なる**。

```text
<demo>/
├── .github/workflows/   CI/CD（インフラ / Jobごと / Functionsごとに分割）
├── terraform/           インフラ定義
├── app/
│   ├── common/          Jobs用の共有ライブラリ
│   ├── run-jobs/{name}/ 1フォルダ = 1 Cloud Run Job（crj-{name}）
│   ├── functions/{name}/1フォルダ = 1 Cloud Functions（cf-{name}）
│   ├── workflows/       Workflows定義（ファイル名＝ワークフロー名）
│   └── queries/         データ品質チェックSQL
├── dbt_project/ または dataform_project/
├── bin/                 ローカル運用スクリプト
├── scripts/             環境構築スクリプト
└── Makefile             コマンドのショートカット
```

**なぜデモを分けているか**: Dataformは指定したGitHubリポジトリを*そのまま*コンパイルし、サブディレクトリを指定できない。dbt構成にフォルダを足すだけでは成立しないため、リポジトリ構成ごと分けている（`50` 章 6節）。

## 3. 前提

* 実装言語は **Python 3.12** を標準とする（取込処理・ユーティリティ）。変換ロジックは SQL（BigQuery標準SQL）。
* IaC は **Terraform**（HCL）を標準とする。手動でのコンソール変更は禁止（`50` 章）。
* リージョンは **`asia-northeast1`（東京）** を既定とし、BigQueryのロケーションは `asia-northeast1` に統一する。
