# 11. BigQuery 命名規則・スキーマ設計

## 1. データセット命名

```text
{layer}_{option_name}
```

| レイヤー | パターン | `{option_name}` の意味 | 例 |
| :--- | :--- | :--- | :--- |
| BRONZE | `bronze_{source_system}` | 連携元システム単位 | `bronze_salesforce`, `bronze_box`, `bronze_pos_api`, `bronze_file_csv` |
| SILVER | `silver_{domain}` | 業務ドメイン単位 | `silver_common`, `silver_sales`, `silver_customer` |
| GOLD | `gold_{consumer}` | 利用者・利用システム単位 | `gold_bi_tool`, `gold_finance`, `gold_ml` |
| 運用 | `ops_{purpose}` | 基盤運用用（ログ・品質チェック結果・メタデータ） | `ops_dataquality`, `ops_audit` |
| 一時 | `tmp_{purpose}` | 検証・移行用。**保持期限（`default_table_expiration`）必須** | `tmp_migration` |

**ルール**

1. **環境コードはデータセット名に含めない**。環境はGCPプロジェクトで分離しているため冗長になる（`01` 章 5節）。
   * 例外：1プロジェクト内に複数環境を同居させざるを得ない場合に限り `{env}_{layer}_{option_name}` を許容し、ADRに理由を記録する。
2. **データセットはIAM境界そのもの**。「この単位で権限を切りたいか」でグルーピングを決める。特にGOLDは、利用システムごとにデータセットを分ける（`30` 章）。
3. ロケーションは全データセットで `asia-northeast1` に統一する。異ロケーション間はJOINできない。
4. `description` に「何のデータか」「オーナーチーム」「連携元」を必ず書く。
5. データセットには `40` 章のラベル（`layer` / `option_name` / `owner` / `managed_by`）を必須で付与する。

## 2. テーブル・ビュー命名

| レイヤー | 種類 | パターン | 例 |
| :--- | :--- | :--- | :--- |
| BRONZE | テーブル | 連携ツールの自動生成名に従う（矯正しない） | `Account`, `sales_transaction` |
| | ビュー | `v_{元テーブル名}` | `v_account_link` |
| SILVER | ディメンション | `d_{データ名}` | `d_account`, `d_product`, `d_calendar` |
| | ファクト | `f_{grain}_{データ名}` | `f_30m_sales_transaction`, `f_1d_sales_actual` |
| | 履歴（SCD Type2） | `d_{データ名}_history` | `d_account_history` |
| GOLD | ビュー（既定） | `v_{分析対象}` | `v_1d_daily_pnl` |
| | 物理テーブル | `t_{分析対象}` | `t_1m_monthly_pnl` |
| 運用 | テーブル | `{purpose}_{対象}` | `dq_result_silver`, `audit_job_run` |

**ルール**

1. `d_`（ディメンション）は時間軸を持たないため `{grain}` を付けない。`f_`（ファクト）は必ず `{grain}` を付ける。
2. `{grain}` は `10` 章の語彙（`30m` / `1h` / `1d` / `1m` / `1y`）のみ。`monthly` などの英単語は使わない。
3. **単数形**で書く（`d_account`。`d_accounts` は不可）。
4. GOLDは**ビューを既定**とし、描画遅延が実測で問題になったものだけを `t_` として物理化する。物理化した場合は必ず元のビュー定義をGitに残す。
5. テーブル名の長さは40文字以内を目安とする。

## 3. 物理設計（義務）

DDLの発行手段（Terraform / Dataform / dbt / 直接SQL）によらず、成果物として満たすべき制約。

### 3.1 パーティション

1. **ファクト（`f_`）はパーティション必須**。基準日列（`business_date` 等）を指定する。
2. **粒度（DAY / MONTH / YEAR）は「保持年数」と「ジョブ上限」から決める**。

   | 上限 | 値 | 日単位での到達時期 |
   | :--- | ---: | :--- |
   | 1テーブルあたりのパーティション数 | 10,000 | 約27年 |
   | **単一ジョブが変更できるパーティション数** | **4,000** | **約11年** |

   後者に到達すると、フルリフレッシュ・洗替・バックフィルが単一ジョブで実行できなくなる（`Too many partitions modified in a single job`）＝**再構築できないデータ基盤**になる。
3. **10年以上保持するテーブルは月単位（`DATE_TRUNC(col, MONTH)`）を既定**とする。
4. **洗替・削除の単位に粒度を合わせる**。月単位で洗い替えるなら月単位パーティション。
5. 月単位にしてもクエリコストは通常変わらない（1クエリの最小課金は10 MB）。粗くしたぶんは基準日列を**第1クラスタ列**に置いて補う。
6. **粒度は後から変更できない**。変更にはテーブル再作成が必要。初期設計で決めること。
7. `require_partition_filter = true` を既定とする（フルスキャン事故の防止）。解除する場合はADRに記録。

### 3.2 クラスタリング

1. **ディメンション（`d_`）はクラスタ必須**。JOINキーとなるID列（`account_id` 等）を指定する。
2. ファクトも推奨。**月単位パーティションの場合は基準日列を第1クラスタ列**に置く。
3. クラスタ列は最大4列。**カーディナリティの低い列を先に**置く（`WHERE` の絞り込み順に合わせる）。

### 3.3 保持・アーカイブ

1. パーティション有効期限は、**10年後の容量を試算し、削減額が運用の複雑さに見合う場合のみ**設定する。BigQueryは90日間更新のないパーティションに長期保存料金（50%引き）を自動適用するため、数GB規模では効果がほぼない。
2. **分析用ファクトに有効期限を設定しない**（過年度比較要件と衝突する）。運用ログなど再現性が不要なテーブルに限る。
3. **GCS上の原本は削除しない**。保管費削減はストレージクラスの段階移行で行う（`13` 章）。
4. 本番のSILVER/GOLDテーブルには `deletion_protection = true`（Terraform）を設定する。

## 4. カラム命名

### 4.1 共通ルール

* 全小文字 `snake_case`。
* **主キー**: `{entity}_id`。単独の `id` は禁止。
* **コード列**: `{対象}_cd` に統一。`_code` は使用しない。

  | サフィックス | 意味 | 型の目安 | 例 |
  | :--- | :--- | :--- | :--- |
  | `_id` | システム内部の識別子（サロゲートキー・連番含む） | INT64 / STRING | `account_id`, `area_id` |
  | `_cd` | 業務上意味のあるコード値（ゼロ埋め等の表記が定まっているもの） | STRING | `area_cd`（`01`〜`10`）, `contract_status_cd` |

  両方を持つ場合は `_id` → `_cd` → 名称 の順に並べる。**BRONZEはソースの列名をそのまま保持**し、リネームはSILVERで行う。
* **真偽値**: `is_` / `has_` で開始（`is_active`, `has_contract`）。NULLを許さず `false` を既定にする。
* **日時列**: 必ず `_utc` または `_jst` を付ける。省略不可。

  | 列名 | 型 | 理由 |
  | :--- | :--- | :--- |
  | `*_at_utc` | `TIMESTAMP` | 絶対時刻。BigQuery内部表現と一致 |
  | `*_at_jst` | `DATETIME` | 壁時計時刻。`TIMESTAMP` を当てると名前と型の意味が矛盾する |
  | `*_date` | `DATE` | 日付のみ。どのタイムゾーンで切ったかを `description` に明記 |

* **数値・金額列**: 単位サフィックス必須（kgとtonの取り違え防止）。複合単位は `{量}_{通貨/単位}_per_{単位}`。
  例: `stock_volume_kg`, `capacity_ton`, `unit_price_jpy_per_kg`
* **件数列**: 集計して生成した件数はGOLDで `_count`（4.3）。
  **ソース側が属性として保持している件数**（Salesforce `NumberOfEmployees` 等）は、
  集計ではないため**SILVERでも `_count` を使ってよい**（`employee_count`）。
  `qty`（数量）は個数・重量などの「量」を指し、件数とは区別する（`10` 章3節）。
* **型の既定**: 整数は `INT64`、小数は原則 `NUMERIC`（金額・数量に `FLOAT64` を使わない）、文字列は `STRING`。真偽値は `BOOL`。
* **`description` 必須**: SILVER/GOLDの全列に日本語で意味・単位・取りうる値を書く。BigQueryのスキーマがデータカタログの正となる。
* **`REQUIRED`（NOT NULL）** は主キーとパーティション列に必ず設定する。

### 4.2 監査項目

トレーサビリティと差分更新のため、各レイヤーに以下の列を必ず付与する。**列名・型・意味は全ツール共通**とし、値の発行方法のみツールごとに規定する（`21` 章）。

| レイヤー | 列名 | 型 | 定義 |
| :--- | :--- | :--- | :--- |
| BRONZE | `_bronze_ingested_at` | TIMESTAMP | BigQueryへの取込日時（UTC）。連携ツールが付与するシステム列（`_fivetran_synced` 等）があれば流用可 |
| | `_bronze_source_uri` | STRING | 取込元のGCSパスまたはAPIエンドポイント（再現性の担保） |
| SILVER | `etl_loaded_at` | TIMESTAMP | 初回挿入日時（UTC） |
| | `etl_updated_at` | TIMESTAMP | 最終更新日時（UTC） |
| | `etl_source_system` | STRING | 由来ソースシステム名（`salesforce` 等） |
| | `etl_batch_id` | STRING | パイプライン実行ID。同一実行内では固定値 |
| | `is_deleted` | BOOL | 論理削除フラグ（`21` 章 CDC） |
| GOLD | `mart_generated_at` | TIMESTAMP | `t_` の物理生成日時（UTC）。`v_` には不要 |
| | `mart_refresh_type` | STRING | `FULL_REFRESH` / `INCREMENTAL`。`t_` のみ |

**運用ルール**

1. **名称固定**: `etl_loaded_at` / `etl_updated_at` に完全固定。`created_date` / `updated_date` / `elt_*` は禁止（ソース側の業務列との混同を招く）。
2. **ソース側の業務日時列とは別列で共存**させる（Salesforce `CreatedDate` → `record_created_at_utc`）。
3. **UTC統一**。JST変換は表示側（ビュー）でのみ行う。
4. 差分処理の基準列は `etl_updated_at`。
5. **実行単位でタイムスタンプを1回だけ確定する**。単一ステートメント内の `CURRENT_TIMESTAMP()` は同一値だが、**ステートメントを跨ぐとずれる**。複数ステートメント構成では直書きを禁止し、実行単位で確定した値（変数／コンパイル変数）を参照する（`21` 章）。
6. **連携ツールがBRONZEのシステム列を付与しない場合**（BigQuery DTS など、GCSを経由しないマネージド転送）、`_bronze_ingested_at` / `_bronze_source_uri` を無理に自前で作らない。取込時刻と実行元の追跡は `ops_{purpose}` の実行履歴テーブルとSILVERの `etl_*` で担保し、**BRONZE監査項目を持たないことと、その代替手段を設計書に明記する**。

### 4.3 派生指標のレイヤー方針

**原則: 派生指標はGOLDでのみ生成する。**

| 記号 | 意味 | 例 |
| :--- | :--- | :--- |
| `kpi_`（接頭） | 経営・事業管理上の最重要指標 | `kpi_gross_profit_jpy` |
| `_rate`（接尾） | 比率・割合（0〜1、%の場合は `_pct`） | `repeat_purchase_rate` |
| `_yoy`（接尾） | 前年同期比 | `sales_volume_yoy` |
| `_diff`（接尾） | 差分（計画と実績など） | `sales_diff_kg` |
| `_count`（接尾） | 件数 | `order_count` |

**SILVERで許容する範囲**（1レコード内で完結する機械的変換のみ）

* 許容: タイムゾーン変換、型キャスト・トリム等のScalar関数、パーティション/クラスタ用の日付・ID列の切り出し、コード値の標準化。
* **禁止**: `GROUP BY` を用いた集約、ウィンドウ関数（移動平均・前日比など）。二重実装リスクとなるため一切禁止。

## 5. ビュー規約

1. GOLDのビューは `SELECT *` を禁止する。列を明示する（上流の列追加で下流が壊れないため）。
2. ビューの多段ネストは**2段まで**。3段以上はテーブル化またはロジック統合を検討する。
3. Authorized View / Authorized Dataset を使い、利用者にSILVERへの直接権限を与えない（`30` 章）。
4. ビュー定義もSQLファイルとしてGit管理し、コンソールでの直接編集は禁止。

## 6. スキーマ変更（後方互換）

| 変更 | 可否 | 手順 |
| :--- | :--- | :--- |
| 列の追加（NULLABLE） | 可 | そのまま追加。既存の下流は影響なし |
| 列のリネーム | 原則不可 | 新列を追加 → 下流を移行 → 旧列を1リリース以上あけて削除 |
| 型の変更 | 原則不可 | 同上（新列を追加して移行） |
| 列の削除 | 条件付き可 | 下流の参照ゼロを確認（`INFORMATION_SCHEMA.VIEWS` / クエリ履歴）してから |
| `REQUIRED` → `NULLABLE` | 可 | — |
| `NULLABLE` → `REQUIRED` | 不可 | テーブル再作成 |
| パーティション粒度の変更 | 不可 | テーブル再作成（`CREATE OR REPLACE ... PARTITION BY`） |

* 破壊的変更は必ず**PRで下流影響（ビュー・BIレポート・外部連携）を列挙**してからマージする。
