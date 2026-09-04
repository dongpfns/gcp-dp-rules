# 50. Git・CI/CD 規約

## 1. リポジトリ構成

| 変換基盤 | リポジトリ | 内容 |
| :--- | :--- | :--- |
| **B: dbt（既定）** | `{system}-platform` | モノレポ1本（`terraform/` `app/` `dbt_project/` `bin/` `scripts/`） |
| **A: Dataform** | `{system}-platform` | 同上（ただしDataform以外） |
| | `{system}-dataform` | Dataform の `definitions/` `includes/` のみ（**必須の分離**） |

モノレポの標準構成:

```text
{system}-platform/
├── .github/workflows/   CI/CD。インフラ / Jobごと / Functionsごとに分ける
├── terraform/           インフラ定義
├── app/
│   ├── common/          Jobs用の共有ライブラリ
│   ├── run-jobs/{name}/ 1フォルダ = 1 Cloud Run Job（crj-{name}）
│   ├── functions/{name}/1フォルダ = 1 Cloud Functions（cf-{name}）
│   ├── workflows/       Workflows定義（ファイル名＝ワークフロー名）
│   └── queries/         データ品質チェックSQL
├── dbt_project/         dbtプロジェクト一式
├── bin/                 ローカル運用スクリプト
├── scripts/             環境構築スクリプト
└── Makefile             コマンドのショートカット
```

* Dataformは**リポジトリの中身をそのままコンパイル**するため、Terraform等の無関係なファイルを混在させられない。よって2リポジトリ構成が必須。
* dbtは任意の場所に置けるためモノレポで完結でき、インフラと変換の変更を1つのPRでレビューできる利点がある。

## 2. ブランチ戦略

| ブランチ | 対応環境 | GCPプロジェクト | マージ元 | 保護 |
| :--- | :--- | :--- | :--- | :--- |
| `main` | 本番 (`prd`) | `{system}-prd` | `staging` からのPRのみ | 必須レビュー2名 / CI必須 / 直push禁止 |
| `staging` | 検証 (`stg`) | `{system}-stg` | `develop` からのPRのみ | 必須レビュー1名 / CI必須 / 直push禁止 |
| `develop` | 開発 (`dev`) | `{system}-dev` | `feature/*` からのPR | 必須レビュー1名 / CI必須 |
| `feature/{issue番号}-{概要}` | 開発 (`dev`) | `{system}-dev` | — | — |
| `hotfix/{issue番号}-{概要}` | 本番 | — | `main` から切り、`main` と `develop` の両方へ | 必須レビュー2名 |

* ブランチ名は `kebab-case`（例: `feature/123-add-box-ingestion`）。
* **本番への反映は `main` へのマージのみ**。ローカルからの `terraform apply` を禁止する。
* `feature` ブランチは1週間以内にマージするか閉じる（長期化した差分は事故のもと）。

## 3. コミット・PR規約

**コミットメッセージ**: Conventional Commits に従う。

```text
feat(silver): d_account に契約状況コードを追加
fix(ingest): Salesforce取込のページング漏れを修正
chore(terraform): providerを6.12へ更新
docs(v2): CDCの範囲条件ルールを追記
```

型: `feat` / `fix` / `refactor` / `perf` / `test` / `docs` / `chore` / `revert`
スコープ: `bronze` / `silver` / `gold` / `ingest` / `terraform` / `workflows` / `dataform` / `dbt` / `ci`

**PR規約**

1. PRテンプレートに以下を必須項目として置く。
   * 変更概要 / 関連Issue
   * **下流影響**（参照しているビュー・BIレポート・外部連携の一覧）
   * **`terraform plan` の差分**（インフラ変更がある場合）
   * **再実行・切り戻し手順**
   * チェックリスト（`90` 章から抜粋）
2. スキーマの破壊的変更を含むPRは、タイトルに `[BREAKING]` を付ける。
3. `CODEOWNERS` を設定し、`terraform/iam.tf` と `schemas/` の変更にはそれぞれセキュリティ担当・データオーナーのレビューを必須にする。

## 4. CI（PR時）

| ワークフローファイル | トリガー | 内容 |
| :--- | :--- | :--- |
| `lint.yaml` | 全PR | `sqlfluff`（BigQuery dialect）/ `ruff` / `black --check` / `yamllint` |
| `security.yaml` | 全PR | `gitleaks`（シークレット検出）/ `tfsec` または `trivy config` |
| `terraform-ci.yaml` | `terraform/**` の変更 | `fmt -check` / `init` / `validate` / `plan`（dev）→ 結果をPRにコメント |
| `python-ci.yaml` | `src/**` の変更 | `pytest`（単体テスト）/ カバレッジ |
| `dataform-ci.yaml` | Dataformリポジトリの全PR | `dataform compile`（コンパイル検証）/ `dataform run --dry-run` |
| `dbt-ci.yaml` | `src/dbt/**` の変更 | `dbt deps` / `dbt compile` / `dbt build --empty`（スキーマ検証） |

* **CIが通らないPRはマージできない**ようブランチ保護で強制する。
* CIは Workload Identity 連携でGCPに接続する。**SAキーJSONをGitHub Secretsに置かない**。

## 5. CD（マージ時）

**デプロイ対象ごとにワークフローを分ける。** 1本にまとめると、無関係な変更でも全体が再デプロイされ、`paths` による変更監視も効かなくなる。

| ワークフローファイル | 監視する `paths` | 内容 |
| :--- | :--- | :--- |
| `tf-apply.yml` | `terraform/**` `app/workflows/**` `app/functions/**` | 対応環境へ `terraform apply`（`main` は手動承認ゲート） |
| `run-job-{name}.yml` | `app/run-jobs/{name}/**` `app/common/**` | コンテナビルド → Artifact Registry へpush（タグ＝コミットSHA）→ 該当Jobのみ更新 |
| `func-{name}.yml` | `app/functions/{name}/**` | 該当Functionのみを `-target` 指定で `terraform apply` |

* **Jobのイメージ更新とインフラの `apply` を衝突させない**。Terraform側は `lifecycle.ignore_changes` でイメージを無視し、タグの管理はCIに任せる。

* **本番（`main`）へのデプロイは手動承認（GitHub Environments の Required reviewers）を必須**にする。
* デプロイ後は自動でスモークテスト（対象Workflowの1回実行、または品質チェックSQLの実行）を走らせる。

## 6. Dataform のデプロイ（基盤Aの注意点）

**Dataformに「デプロイ」工程は存在しない。** DataformリポジトリはGitHubリポジトリを直接参照し、指定ブランチ/タグの内容をその場でコンパイルして実行する。

1. **Terraform** が `google_dataform_repository` と Release Config（`rc-{env}`）を作り、環境ごとに参照ブランチを紐付ける。
   * `rc-prd` → `main` / `rc-stg` → `staging` / `rc-dev` → `develop`
2. **GitHub Actions** の役割は**デプロイではなく検証（コンパイル＋dry-run）**。
3. GitHubへマージした時点で、次回のコンパイル対象が新しいコードになる。
4. **GitHubトークンの受け渡しが最も詰まる箇所**:
   * Dataformのサービスエージェントを明示的に作成する（API有効化直後は存在せずIAM付与が失敗する）。
   * トークンは Secret Manager に置き、サービスエージェントに `secretAccessor` を付与する。
   * トークンは**当該Dataformリポジトリの読み取り権限のみ**（またはGitHub App）で発行する。
5. **Workflowsから起動する**（`21` 章 5.3）。Dataform内蔵スケジュールでは監査項目用の `vars` を実行ごとに変えられない。

## 7. リリース・タグ

* 本番リリースごとに `v{YYYY.MM.DD}-{連番}` のタグを打つ（例: `v2026.09.04-1`）。
* リリースノートに「変更されたテーブル」「破壊的変更の有無」「切り戻し手順」を書く。
* 切り戻しは**タグを指定して再デプロイ**する。`git revert` によるコード戻しとインフラの戻しは別物なので、両方の手順を書く。

## 8. .gitignore（必須項目）

```text
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.secret.tfvars

# 認証情報
*.json.key
credentials*.json
.env

# dbt
target/
dbt_packages/
logs/

# Dataform
.df-credentials.json
node_modules/

# Python
__pycache__/
.venv/
.pytest_cache/
```
