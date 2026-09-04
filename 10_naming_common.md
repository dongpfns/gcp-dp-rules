# 10. 命名の共通原則

すべてのリソース・コード・ファイルに適用する最上位の命名ルール。個別章（`11`〜`13`）はこれを具体化するものであり、矛盾した場合は本章を正とする。

## 1. 記法（ケース）の使い分け

| 対象 | 記法 | 例 |
| :--- | :--- | :--- |
| BigQuery データセット / テーブル / ビュー / カラム | `snake_case`（全小文字） | `silver_common`, `d_account`, `account_id` |
| GCPリソース名（GCS / Cloud Run / Functions / Workflows / Scheduler / Pub/Sub / Secret / Artifact Registry / DTS） | `kebab-case`（全小文字） | `example-prd-raw`, `cf-ingest-salesforce` |
| Terraform の識別子（resource名 / variable / module / output） | `snake_case` | `resource "google_bigquery_table" "d_account"` |
| Python（モジュール / 関数 / 変数） | `snake_case`、クラスは `PascalCase` | `ingest_salesforce.py`, `class SalesforceClient` |
| ディレクトリ・SQL/SQLX/YAMLファイル名 | 中身のリソース名と**完全一致**させる | `wf-main-daily.yaml` ↔ Workflows名 `wf-main-daily` |
| 環境変数 | `UPPER_SNAKE_CASE` | `GCP_PROJECT_ID`, `ETL_BATCH_ID` |
| GitHub Actions ワークフローファイル | `kebab-case` + `.yaml`（`.yml` 禁止） | `terraform-ci.yaml` |

> **なぜ2種類あるのか**: BigQueryのオブジェクト名はハイフンを使えず（クォートが必要になる）、逆にGCSバケット名はアンダースコアを使えない。サービス側の制約に合わせた結果であり、恣意的な使い分けではない。**「BigQueryの中は `_`、BigQueryの外は `-`」**と覚える。

## 2. 共通の構成要素

命名に使う語は下表の語彙に統一し、同義語の混在を禁止する。

| プレースホルダ | 意味 | 取りうる値の例 |
| :--- | :--- | :--- |
| `{system}` | システム（プロダクト）識別子。GCPプロジェクトIDの接頭辞 | `example` |
| `{env}` | 環境コード | `prd` / `stg` / `dev` |
| `{layer}` | データレイヤー | `bronze` / `silver` / `gold` |
| `{source_system}` | 連携元システム | `salesforce` / `box` / `pos_api` / `file_csv` |
| `{domain}` | 業務ドメイン（SILVERのグルーピング単位） | `common` / `sales` / `customer` |
| `{consumer}` | 利用者・利用システム（GOLDのグルーピング単位） | `bi_tool` / `finance` / `ml` |
| `{grain}` | ファクトの集計粒度 | `30m` / `1h` / `1d` / `1m`（＝1か月） |
| `{freq}` | 実行頻度 | `hourly` / `daily` / `weekly` / `monthly` |

## 3. 用語の統一（同義語の禁止）

| 採用する語 | 使用禁止の同義語 | 備考 |
| :--- | :--- | :--- |
| `cd`（コード値） | `code`, `kbn`, `type_code` | `_cd` に統一 |
| `id`（識別子） | `no`, `key`, `seq_no`, `number` | 単独の `id` も禁止（必ず `{entity}_id`） |
| `qty`（数量） | `quantity` | 個数・重量・体積などの「量」。**件数は `_count` を使う**（`11` 章4.1） |
| `amt`（金額） | `amount`, `price`（単価は `unit_price_`） | 通貨サフィックス必須 |
| `nm` は使わない → `name` | `nm`, `title`（表示名は `_display_name`） | |
| `dt` は使わない → `_at_utc` / `_at_jst` / `_date` | `dt`, `datetime`, `timestamp` | 日付のみは `_date` |
| `flg` は使わない → `is_` / `has_` | `flg`, `flag`, `_fl` | |
| `etl_`（基盤側の監査項目） | `elt_`, `meta_`, `sys_` | `11` 章 4.2 |

## 4. 略語ルール

* 略語は**下表の許可リストに載っているものだけ**を使う。載っていない語は略さずに書く。
* 追加したい略語はPRで許可リストに追記してからコード側で使う。

| 略語 | 正式 | 略語 | 正式 |
| :--- | :--- | :--- | :--- |
| `cd` | code | `qty` | quantity |
| `id` | identifier | `amt` | amount |
| `avg` | average | `max` / `min` | maximum / minimum |
| `ds` | dataset | `ym` | year-month |
| `tz` | timezone | `pk` | primary key |
| `sa` | service account | `sla` / `slo` | service level agreement / objective |

**禁止する略し方**: 日本語ローマ字の略（`kbn`, `tanka`, `uriage`）、母音抜き（`prdct`, `cstmr`）、社内の口語略（`ジェネ`, `トラン`）。

## 5. 接頭辞・接尾辞の一覧（横断）

| 位置 | 記号 | 意味 | 適用対象 |
| :--- | :--- | :--- | :--- |
| 接頭 | `d_` | ディメンション（マスタ） | SILVERテーブル |
| 接頭 | `f_` | ファクト（数値・時系列） | SILVERテーブル |
| 接頭 | `v_` | ビュー | 全レイヤー |
| 接頭 | `t_` | 物理化テーブル（キャッシュ） | GOLD |
| 接頭 | `stg_` | 一時ステージング（実行内で完結、保持しない） | 全レイヤー |
| 接頭 | `kpi_` | 経営・事業KPI列 | GOLDカラム |
| 接頭 | `is_` / `has_` | 真偽値 | カラム |
| 接尾 | `_id` / `_cd` | 識別子 / 業務コード | カラム |
| 接尾 | `_utc` / `_jst` | タイムゾーン | 日時カラム |
| 接尾 | `_date` | 日付（時刻を持たない） | カラム |
| 接尾 | `_kg` / `_ton` / `_jpy` / `_jpy_per_kg` | 単位 | 数値カラム |
| 接尾 | `_rate` / `_yoy` / `_diff` | 派生指標 | GOLDカラム |
| 接尾 | `_count` | 件数 | GOLDの派生指標。ソースが属性として持つ件数はSILVERでも可（`11` 章4.1） |

## 6. 全体で禁止すること

1. **環境名をリソース名の途中に埋め込まない**。環境は必ずGCPプロジェクトで分け、名前に含める場合は `{system}-{env}-...` のように先頭側に置く。
2. **日付・バージョンをテーブル名に付けない**（`d_account_20260901`, `d_account_v2`）。世代管理はパーティションとGitで行う。バックフィル用の一時テーブルは `stg_` を付け、実行後に必ず削除する。
3. **個人名・チーム名をリソース名に入れない**（`d_account_tanaka`）。
4. **日本語・全角文字を識別子に使わない**。日本語は `description` / ラベルの値ではなくドキュメントに書く。
5. **`tmp` / `test` / `bk` / `old` / `new` を本番リソース名に使わない**。
6. **予約語・BigQuery組込み関数名を列名にしない**（`from`, `select`, `date`, `time`, `hash`）。
