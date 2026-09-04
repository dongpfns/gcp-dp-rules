# 13. Cloud Storage バケット構成・パス規約

## 1. バケット構成

バケット名はGCP全体でグローバル一意。**`{system}-{env}-{purpose}`** とする（アンダースコア不可）。

| バケット | 用途 | ストレージクラス | バージョニング | ライフサイクル |
| :--- | :--- | :--- | :--- | :--- |
| `{system}-{env}-tfstate` | Terraform state | Standard | **有効（必須）** | 旧バージョン90日で削除 |
| `{system}-{env}-raw` | 外部連携ファイルの着地点（生ファイル） | Standard | 有効 | 90日でNearline → 365日でColdline |
| `{system}-{env}-archive` | BRONZE再構築用の原本長期保管 | Nearline | 有効 | 365日でColdline → 1095日でArchive |
| `{system}-{env}-functions-src` | Cloud Functions のソースZIP | Standard | 有効 | 旧バージョン30日で削除 |
| `{system}-{env}-artifacts` | dbt docs / データ辞書などの生成物 | Standard | 無効 | 30日で削除 |
| `{system}-{env}-tmp` | 一時作業領域（BQ export の中間ファイル等） | Standard | 無効 | **7日で削除** |
| `{system}-{env}-export` | 外部システムへの受け渡し | Standard | 有効 | 30日で削除 |

**ルール**

1. **`Delete` ライフサイクルはこの表にある `tmp` / `artifacts` / `export` / 旧バージョンのみ**。`raw` / `archive` の現行オブジェクトを削除するルールは書かない。原本は再構築の唯一の手段。
2. 保管費を下げたいときは **削除ではなく `SetStorageClass`**（Standard → Nearline → Coldline → Archive）。パスもAPIも変わらないため再取込処理の変更が不要。
3. `uniform_bucket_level_access = true` を必須とする（オブジェクト単位ACLを使わない）。
4. `public_access_prevention = "enforced"` を必須とする。
5. ロケーションは `asia-northeast1`（BigQueryと同一）。跨ぐとロード時に追加コストと制約が発生する。
6. 本番バケットは Terraform で `force_destroy = false`。
7. 暗号化はGoogle管理鍵を既定とし、要件がある場合のみCMEK（`30` 章）。

## 2. オブジェクトパス規約

### 2.1 raw（取込）

```text
gs://{system}-{env}-raw/{source_system}/{object}/dt={YYYY-MM-DD}/{run_id}/{object}_{YYYYMMDDTHHMMSSZ}.{ext}
```

例:
```text
gs://example-prd-raw/salesforce/account/dt=2026-09-04/20260904T060000Z-a1b2c3d/account_20260904T060012Z.json
gs://example-prd-raw/file_csv/sales_actual/dt=2026-09-04/20260904T060000Z-a1b2c3d/sales_actual_20260904T060030Z.csv
```

| 要素 | 規約 |
| :--- | :--- |
| `{source_system}` | BRONZEデータセット `bronze_{source_system}` と一致させる |
| `{object}` | ソース側のオブジェクト名（snake_case化）。BRONZEテーブル名と対応させる |
| `dt={YYYY-MM-DD}` | **Hiveパーティション形式**。BigQuery外部テーブル・ロード時のパーティション認識に使う。日付は「データの対象日」であり、処理実行日ではない |
| `{run_id}` | パイプライン実行ID（`ETL_BATCH_ID` と同値）。再実行時の切り分けに使う |
| ファイル名 | 末尾にUTCタイムスタンプ（ISO8601 basic）を付け、同一実行内の上書きを防ぐ |

* **既存オブジェクトを上書きしない**。再実行は新しい `{run_id}` の下に書く。
* 圧縮する場合の拡張子は `.jsonl.gz` / `.csv.gz`。BigQueryロードは `gzip` のみ対応。
* 1ファイルのサイズは **128MB〜1GB** を目安に分割する（小さすぎるファイルが大量にあるとロードが遅い）。

### 2.2 archive（原本保管）

```text
gs://{system}-{env}-archive/{source_system}/{object}/dt={YYYY-MM-DD}/{元ファイル名}
```

* raw から一定期間後に移送する、または raw 自体をライフサイクルでクラス移行して兼ねる（どちらかに統一し、両方は作らない）。
* サンプル実装では **raw のライフサイクルでクラス移行して兼ねる**方式を採用し、`archive` バケットは大量データ・法定保存要件がある場合のみ追加する。

### 2.3 export（外部連携）

```text
gs://{system}-{env}-export/{consumer}/{object}/dt={YYYY-MM-DD}/{object}_{YYYYMMDD}_{seq}.csv
```

* BigQueryの `EXPORT DATA` は複数ファイルに分割されるため、ワイルドカード `_*.csv` を前提としたファイル名にする。
* 受け渡し完了の合図が必要な場合は、同ディレクトリに `_SUCCESS` ファイルを最後に置く。

### 2.4 tmp

```text
gs://{system}-{env}-tmp/{run_id}/...
```

* 実行IDごとにディレクトリを切る。ライフサイクル7日で自動削除されるため、後片付けコードは不要。

## 3. BigQueryとの接続方法

| 用途 | 方式 | 使い分け |
| :--- | :--- | :--- |
| 定期的な取込（既定） | **ロードジョブ**（`bq load` / Python Client） | BRONZEに実体を持つ。以後のクエリが速く安い |
| アドホック・調査 | 外部テーブル（`CREATE EXTERNAL TABLE`） | 実体を持たないためロード不要。都度GCSを読むので遅い |
| 大量の履歴ファイル | BigLake外部テーブル + Hiveパーティション | メタデータキャッシュで高速化できる |

* **本番のBRONZEはロードジョブで実体化する**を既定とする。外部テーブルのみに依存すると、GCS側のファイル削除・移動でクエリが壊れる。
* ロード時は `write_disposition` を `20` 章の取込パターンに従って明示する（既定任せにしない）。
* スキーマ自動検出（`autodetect`）は**開発時のみ**。本番はスキーマJSONを明示する（型が実行ごとに変わる事故を防ぐ）。
