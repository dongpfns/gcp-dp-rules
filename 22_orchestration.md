# 22. オーケストレーション規約

## 1. 構造（原則）

```text
Cloud Scheduler  ──起動──►  Cloud Workflows  ──呼出──►  Cloud Functions（取込）
（定時 / JST）                （唯一の司令塔）  ──呼出──►  BigQuery DTS（取込）
                                             ──呼出──►  Dataform / Cloud Run Jobs（変換）
                                             ──呼出──►  BigQuery（品質チェック）
                                             ──通知──►  Pub/Sub → 通知チャネル
```

**原則**

1. **順序・分岐・リトライ・エラー処理は Workflows に集約する。** Functions から次のFunctionsを呼ぶ、SQL の最後で次のジョブを起動する、といった暗黙の連鎖を禁止する（依存関係が不可視になり、障害時に追えない）。
2. **Scheduler は Workflows しか起動しない。** 個別のFunctions/Jobsを直接叩かない。
3. **Workflows は「順序制御」だけを担う。** ビジネスロジック（データの加工判断）をWorkflowsのYAMLに書かない。
4. **1パイプライン = 1 Workflow。** テーブル単位でWorkflowを増やさない。

## 2. Workflows 実装規約

### 2.1 標準ステップ構成

```text
init          run_id / etl_ts の採番（実行単位で1回だけ確定）
  ↓
ingest        取込（並列可）      → 失敗時 notify_failure
  ↓
check_bronze  BRONZE品質チェック  → NGなら abort
  ↓
transform     SILVER変換          → 失敗時 notify_failure
  ↓
assert        SILVER品質チェック  → NGなら abort
  ↓
mart          GOLD生成
  ↓
notify_success / notify_failure
```

### 2.2 ルール

1. **`run_id` と `etl_ts` は Workflows の先頭で1回だけ採番**し、以降の全ステップに引き渡す。各ステップで個別に `CURRENT_TIMESTAMP()` を取らない（`11` 章 4.2 ルール5）。
   * 形式: `run_id = {YYYYMMDD}T{HHMMSS}Z-{workflow名}`（例: `20260904T060000Z-wf-main-daily`）
2. **リトライは Workflows の `retry` で定義**する。既定は「指数バックオフ・最大3回・初回10秒」。
   * 冪等でないステップ（追記型のロード等）にリトライを設定しない。
3. **タイムアウトを全ステップに設定**する。無指定を禁止。
4. **エラーハンドラを必ず持つ**（`try/except`）。失敗時は Pub/Sub に構造化メッセージを publish し、通知チャネルへ流す。
5. **ステップ名は `snake_case` の動詞始まり**（`ingest_salesforce`, `run_dataform`, `check_bronze_freshness`）。
6. **長時間処理は非同期起動＋ポーリング**にする。BigQueryジョブ・Dataform invocation・Cloud Run Jobs は起動APIと状態取得APIを分けて呼ぶ。
7. Workflowsのログには `run_id` を必ず含める。
8. **並列実行（`parallel`）は独立した取込にのみ使う**。変換は依存関係があるため直列またはツール側のDAGに委ねる。
9. **YAMLとして壊れない書き方をする**。プレーンスカラーは `": "` を含められないため、以下は**不正なYAMLになる**（デプロイ時まで気づきにくい）。
   * 式を複数行に折り返す（`json.encode({...})` を改行して書く）→ 値は `assign` でマップとして定義し、呼び出し側は1行の式にする。
   * 式の中の文字列に `": "` が入る（`"failed: " + state`）→ 値全体をシングルクォートで囲む。
   * CIで `yamllint` またはYAMLパーサによる構文チェックを必ず通す。
10. **Terraformの `templatefile()` で読む場合、Workflows自身の式は `$${...}` とエスケープする**。忘れるとplan時にエラーになるか、意図しない空文字が埋め込まれる。

### 2.3 二重起動の防止

* Scheduler のリトライ回数は **0** にする（Workflows側でリトライするため）。
* Workflows の先頭で「同一パイプラインが実行中でないか」を確認する。実行中なら `skip` して正常終了させ、その旨を通知する。
  * 実装: 実行状態を BigQuery の `ops_audit.job_run` テーブル、または GCS のロックオブジェクトで管理する。

## 3. Cloud Functions 実装規約（取込）

1. Gen2、Python 3.12、リージョン `asia-northeast1`。
2. エントリポイント関数名は `main`。HTTPトリガーを既定とし、Workflowsから呼ぶ。
3. **タイムアウトは実測の3倍**を設定する（上限60分）。9分を超える見込みなら Cloud Run Jobs にする。
4. メモリ・CPUは実測に基づき明示する（既定値任せにしない）。
5. `min_instances = 0`（コスト）、`max_instances` は連携先のレート制限から逆算して設定する。
6. **サービスアカウントは関数ごとに分けず、役割ごとに分ける**（`30` 章）。
7. 依存は `requirements.txt` にバージョン固定（`==`）で書く。
8. ソースは GCS（`{system}-{env}-functions-src`）にZIPで配置し、Terraformが参照する。ZIPのハッシュを `object` 名に含めて更新を検知させる。
9. **共有ライブラリに依存させない**。Functionsは単一ディレクトリをZIP化してデプロイするため、自己完結の実装にしておくと配布の仕組みが不要になる。共有ライブラリが必要な処理は Cloud Run Jobs にする。

## 4. Cloud Run Jobs 実装規約

1. dbt実行および長時間・大容量の取込に使う。ソース配置は `app/run-jobs/{name}/`（`12` 章 2節）。
2. **イメージタグはコミットSHA固定**（`12` 章 4節）。ビルドコンテキストはリポジトリルートにし、共有ライブラリ（`app/common/`）をイメージに同梱する。
3. `task_count = 1` を既定。並列化する場合は `CLOUD_RUN_TASK_INDEX` を使い、分割単位を明示する。
4. `max_retries = 0` にし、リトライはWorkflows側で制御する（二重に効くと制御不能になる）。
5. 実行パラメータは環境変数のオーバーライド（`overrides`）でWorkflowsから渡す。

## 5. BigQuery Data Transfer Service

1. **Workflows から手動起動（`StartManualTransferRuns`）を既定**とし、完了をポーリングして後続に進む。
2. DTS内蔵スケジュールを使ってよいのは、後続処理が不要な単純同期のみ。
3. 転送設定はTerraformで管理する（コンソール作成禁止）。
4. 通知メール設定は使わず、`60` 章の統一通知経路に載せる。

## 6. スケジュール設計

| 種別 | 既定の時刻（JST） | 備考 |
| :--- | :--- | :--- |
| 日次 | 05:00 | ソース側の締め時刻＋バッファ2時間以上を確保する |
| 時次 | 毎時 10分 | 毎時00分は他システムと衝突しやすいため避ける |
| 月次 | 翌月1日 06:00 | 日次の完了後に走らせる |

* **タイムゾーンは `Asia/Tokyo` に統一**し、cron式はJSTで書く。UTCとの混在を禁止する。
* 依存関係のあるパイプラインを「時刻をずらして待つ」設計にしない。**先行の完了イベント（Pub/Sub）で起動する**か、1本のWorkflowに統合する。

## 7. 通知

| イベント | 通知先 | 内容 |
| :--- | :--- | :--- |
| 成功 | 通知しない（日次サマリのみ） | ノイズを減らす |
| 失敗 | チャット + メール（即時） | `run_id` / ワークフロー名 / 失敗ステップ / エラー要約 / ログURL |
| 品質チェックNG | チャット（即時） | 対象テーブル / チェック名 / 実測値 / 閾値 |
| 遅延（SLO超過） | チャット | 想定完了時刻と現在の状態 |

* 通知本文には**必ず `run_id` とCloud LoggingへのURL**を含める。
* 通知先のWebhook URLは Secret Manager で管理する。
