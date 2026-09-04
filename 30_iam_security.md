# 30. IAM・セキュリティ規約

## 1. 原則

1. **最小権限**: 必要なリソースに必要なロールのみ。プロジェクトレベルの `roles/bigquery.dataEditor` のような広い付与を禁止する。
2. **基本ロールの禁止**: `roles/owner` / `roles/editor` / `roles/viewer` を人にもSAにも付与しない（ブートストラップの初回のみ例外、ADRに記録）。
3. **人にはグループ、システムにはサービスアカウント**。個人アカウントへの直接付与を禁止する。
4. **IAMはすべてTerraform管理**。コンソールでの手動付与を禁止する（ドリフト検知の対象）。
5. **SAキー（JSONキーファイル）を発行しない**。GCP内はSAの直接アタッチ、外部からはWorkload Identity連携を使う。

## 2. サービスアカウント設計

**役割ごとに分ける**（リソースごとに分けない）。数が増えすぎると権限の全体像が追えなくなる。

| SA名 | 用途 | 主なロール |
| :--- | :--- | :--- |
| `sa-ingest` | 取込処理（Functions / Run Jobs / DTS） | GCS: raw への `objectAdmin`<br>BQ: `bronze_*` データセットへ `dataEditor`<br>Secret: 該当シークレットの `secretAccessor` |
| `sa-transform` | 変換（Dataform / dbt ランナー） | BQ: `bronze_*` へ `dataViewer`、`silver_*` / `gold_*` へ `dataEditor`<br>プロジェクト: `bigquery.jobUser` |
| `sa-workflow` | Workflows 実行 | 各SAへの `iam.serviceAccountUser`<br>Functions/Run の `invoker`<br>BQ: `jobUser`（品質チェック用）<br>Pub/Sub: `publisher` |
| `sa-scheduler` | Scheduler → Workflows 起動 | `workflows.invoker` のみ |
| `sa-terraform` | CI/CDからのインフラ適用 | 必要なリソース管理ロール（`50` 章、Workload Identity連携で使用） |
| `sa-bi` | BIツールからの参照 | BQ: `gold_*` へ `dataViewer`、プロジェクト `jobUser` |

**ルール**

* **1SA = 1責務**。取込SAが変換もできる、という状態を作らない。
* **`sa-transform` に `bronze_*` の書き込み権限を与えない**。取込と変換の責務を物理的に分ける。
* SAの `description` に「誰が・何のために使うか」を必ず書く。
* SAの権限変更は必ずPRレビューを通す（CODEOWNERSでセキュリティ担当を必須レビュアにする）。

## 3. データアクセス境界

```text
                      bronze_*      silver_*      gold_*
sa-ingest             編集          －            －
sa-transform          参照          編集          編集
sa-bi / BIツール      －            －            参照（Authorized View経由）
分析者（グループ）    －            参照          参照
データ基盤開発者      参照          参照          参照   （本番は参照のみ。変更はCI/CD経由）
```

**ルール**

1. **BIツール・業務ユーザーにSILVERへの直接権限を与えない**。GOLDの Authorized View / Authorized Dataset 経由でのみ公開する。
2. **GOLDは利用者ごとにデータセットを分ける**（`gold_bi_tool` / `gold_finance` / `gold_ml`）。データセットがそのままIAM境界になる。
3. **本番環境で人が書き込み権限を持たない**。緊急時は「一時的に付与し、作業後に剥がし、記録を残す」break-glass手順をドキュメント化しておく。
4. 開発環境（`dev`）は開発者に編集権限を与えてよい。ただしIAM定義はTerraformに書く。

## 4. 列・行レベルのアクセス制御

| 手段 | 使う場面 |
| :--- | :--- |
| **Authorized View** | 列の制限・行の絞り込みを行い、基テーブルの権限を渡さない（既定の手段） |
| **Policy Tag（列レベル）** | 個人情報列を、同一テーブル内で権限により隠す必要がある場合 |
| **Row Access Policy** | 部門・拠点ごとに同一テーブルの行を出し分ける場合 |
| **Data Masking** | 権限のない利用者にハッシュ値・NULLを見せる場合 |

* まずは Authorized View で設計する。Policy Tag / Row Access Policy は管理コストが高いため、要件が明確な場合のみ導入する。
* 個人情報を含む列は、**そもそもSILVERに置かない／取込時にハッシュ化する**ことを先に検討する（`20` 章 ETL例外1）。

## 5. Secret 管理

1. 認証情報・APIキー・Webhook URLは **Secret Manager** に格納する。
2. **Terraformでシークレットの「入れ物」（`google_secret_manager_secret`）は作るが、「値」（`secret_version`）は投入しない**。値をtfstateに残さないため、投入は `gcloud` またはコンソールで手動／別経路で行う。
3. シークレットのバージョンは `latest` ではなく**明示バージョンを参照**することを推奨する（ローテーション時の事故防止）。ただし自動ローテーション運用がある場合は `latest`。
4. アクセス権は**シークレット単位で付与**する（プロジェクトレベルの `secretAccessor` を禁止）。
5. ローテーション周期を各シークレットの `description` に記載する。

## 6. コード・リポジトリ上の禁止事項

* 認証情報・トークン・実データのコミット。CIでシークレットスキャン（`gitleaks` 等）を必須にする。
* `*.tfvars` のうち機密を含むものは `.gitignore` 対象とし、`*.secret.tfvars` の命名で明示する。
* サンプルデータをコミットする場合は**必ず匿名化・合成データ**とする。

## 7. ネットワーク・その他

* Cloud Functions / Cloud Run は原則 `ingress = INTERNAL_AND_CLOUD_LOAD_BALANCING`（外部からの直接呼び出しを禁止）とし、Workflowsからの呼び出しのみ許可する。
* 外部APIへのアウトバウンドで送信元IP固定が必要な場合は VPCコネクタ + Cloud NAT を使う。
* BigQueryの `VPC Service Controls` は、機密データを扱う場合に検討する（導入すると開発の手間が増えるため、要件を確認してから）。
* 監査ログ（Data Access ログ）はBigQueryへエクスポートし、`ops_audit` データセットに保管する（`60` 章）。
