# 21. 変換実装規約（SILVER / GOLD）

`11` 章の命名・スキーマ規約を、実際の変換コードでどう満たすかを定める。

## 1. 共通規約（ツール非依存）

1. **変換はSQLで書く**。Pythonで行単位に処理しない。
2. **BRONZEを直接参照してよいのはSILVERのみ**。GOLDはSILVERのみを参照する。
3. **1モデル＝1テーブル/ビュー**。1ファイルに複数のDDL/DMLを書かない。
4. **`SELECT *` 禁止**（BRONZEからの取り出しを含む）。列を明示する。
5. **ハードコード禁止**: プロジェクトID・データセット名は変数／参照関数（`${ref()}` / `ref()`）で解決する。
6. **NULL安全な比較**: 値の変更判定には `IS DISTINCT FROM` を使う。`!=` はNULLを含む比較で正しく動作しない。
7. **監査項目は共通関数／マクロで付与する**。各モデルで手書きしない。
8. **タイムスタンプは実行単位で1回だけ確定**（`11` 章 4.2 ルール5）。`CURRENT_TIMESTAMP()` の直書きを禁止する。
9. **`etl_loaded_at` を `MERGE` の UPDATE 対象から外す**。`MERGE` は既定で全列を更新するため、外さないと実行のたびに「初回挿入日時」が上書きされ、列の意味が失われる。
   * dbt: `merge_exclude_columns = ['etl_loaded_at']`
   * Dataform: 除外指定がないため、`incremental()` 時に自テーブルへ `LEFT JOIN` して既存値を `COALESCE` で引き継ぐ（下記5.5）。
10. SQLの整形は `sqlfluff`（dialect: `bigquery`）で統一し、CIで検査する。

## 2. SILVER層の実装

### 2.1 標準の変換手順

```text
BRONZE（生データ）
  ↓ 1. 列の選択と改名   （AreaCode → area_cd）
  ↓ 2. 型の統一         （STRING の "1" → INT64、金額は NUMERIC）
  ↓ 3. 値の標準化       （トリム、全半角、コード値のゼロ埋め、NULL表現の統一）
  ↓ 4. タイムゾーン付与 （*_at_utc: TIMESTAMP / *_at_jst: DATETIME）
  ↓ 5. 単位の明示       （volume → volume_kg）
  ↓ 6. 監査項目の付与   （etl_loaded_at / etl_updated_at / etl_source_system / etl_batch_id / is_deleted）
  ↓ 7. 重複排除         （P3の場合のみ QUALIFY で最新行）
SILVER
```

* **禁止**: `GROUP BY` を伴う集約、ウィンドウ関数による派生指標（移動平均・前日比）、複数ドメインを跨ぐ業務ロジック。これらはGOLDで行う。
* 許容されるウィンドウ関数は **重複排除の `ROW_NUMBER()` のみ**。

### 2.2 materialization の選択

| 対象 | 既定 | 条件 |
| :--- | :--- | :--- |
| `d_`（ディメンション） | incremental（`MERGE`） | 件数が10万行未満なら table（全件洗替）でもよい。単純なほうが事故が少ない |
| `f_`（ファクト） | incremental（`MERGE`、パーティション範囲限定） | 必須 |
| SILVERの中間 | 作らない | どうしても必要なら ephemeral / CTE で表現し、実体を残さない |

## 3. GOLD層の実装

1. **既定はビュー（`v_`）**。実測で描画遅延が問題になったものだけ `t_` として物理化する。
2. 物理化する場合は**全件洗替（`CREATE OR REPLACE TABLE`）を既定**とする。GOLDは再生成可能であるべきで、差分更新にすると不整合の原因になる。全件洗替が重すぎる場合のみ、パーティション単位の洗替を検討する。
3. 派生指標の命名は `11` 章 4.3 に従う（`kpi_` / `_rate` / `_yoy` / `_diff` / `_count`）。
4. 同じ指標を複数のGOLDテーブルで別々に計算しない。共通指標は1つのGOLDモデルにまとめ、他はそれを参照する。
5. BIツールへは **Authorized View 経由**で公開し、SILVERへの直接権限は与えない（`30` 章）。

## 4. CDC（差分検知・論理削除）

### 4.1 適用範囲

「物理削除検知（論理削除フラグ更新）」は、**対象範囲においてBRONZEがソース側の完全な状態を毎回洗い替えている場合にのみ適用できる**（`20` 章 P1 / P2 / P5）。

| 取込パターン | 削除検知 |
| :--- | :--- |
| P1 全件洗替 | 可（テーブル全体） |
| P2 範囲洗替 | 可（**範囲内のみ**） |
| P3 差分追記 / P4 差分マージ | **不可**。ソース側の削除フラグ／削除イベントを受け取る設計にする |
| P5 CDCログ | 可（削除イベントで検知） |

### 4.2 ロジック（概念）

1. SILVERに対して `MERGE` を実行する。
2. **新規・更新**: ソースにキーが存在すれば値を最新化し、`is_deleted = FALSE` にする（復活を含む）。
3. **削除検知**: SILVERに存在しソースに存在せず、かつ `is_deleted = FALSE` の行を `is_deleted = TRUE` に更新する。既に論理削除済みの行はスキャン対象から外す。
4. 値の変更判定は `IS DISTINCT FROM`。
5. **物理削除はしない**。ダウンストリームは `WHERE is_deleted = FALSE` でフィルタする。

### 4.3 範囲限定型（P2）の必須ルール

**範囲条件は同一の式を3か所すべてに書く。**

```sql
-- ★ 範囲は「実行単位で確定した変数」で持ち、3か所で同じ変数を参照する
DECLARE window_start DATE DEFAULT DATE_TRUNC(DATE_SUB(CURRENT_DATE('Asia/Tokyo'), INTERVAL 14 MONTH), MONTH);
DECLARE etl_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP();
DECLARE batch_id STRING DEFAULT @run_id;

MERGE `silver_sales.f_30m_sales_transaction` AS tgt
USING (
  SELECT ...
  FROM `bronze_pos_api.sales_transaction`
  WHERE business_date >= window_start          -- ① ソース側の抽出
) AS src
ON  tgt.business_date >= window_start           -- ② 結合条件（コスト最適化：パーティションプルーニング）
AND tgt.business_date = src.business_date
AND tgt.store_cd      = src.store_cd
AND tgt.slot_started_at_jst = src.slot_started_at_jst
WHEN MATCHED AND (
      tgt.shipment_volume_ton IS DISTINCT FROM src.shipment_volume_ton
   OR tgt.is_deleted IS DISTINCT FROM FALSE
) THEN UPDATE SET
      shipment_volume_ton = src.shipment_volume_ton,
      is_deleted          = FALSE,
      etl_updated_at      = etl_ts,
      etl_batch_id        = batch_id
WHEN NOT MATCHED BY TARGET THEN INSERT (...) VALUES (...)
WHEN NOT MATCHED BY SOURCE
     AND tgt.business_date >= window_start      -- ③ 削除検知（誤削除防止）
     AND tgt.is_deleted = FALSE
THEN UPDATE SET is_deleted = TRUE, etl_updated_at = etl_ts, etl_batch_id = batch_id;
```

| 書く場所 | 目的 | 抜けたときに起きること |
| :--- | :--- | :--- |
| ① ソース側 `WHERE` | ターゲットと範囲を揃える | 範囲外の行が `NOT MATCHED` → **INSERTされ重複** |
| ② `ON` 句 | パーティションプルーニング | スキャンコストが下がらない |
| ③ `WHEN NOT MATCHED BY SOURCE` | 誤削除の防止 | 範囲外の**過去データが一括で論理削除** |

**追加ルール**

* ウィンドウは**ソース側の処理単位の境界に丸める**（月抽出なら `DATE_TRUNC(..., MONTH)`）。
* ウィンドウ幅は**ソースが遡って訂正しうる最大期間より長く**取る。
* **取得側（`20` 章の取込範囲）とウィンドウ幅を同じ期間に揃える**。ずれていること自体が設計欠陥。
* **ウィンドウ外に更新が届いたことを検知するアサーションを常設**し、検知したらフルリフレッシュを手動判断する。

### 4.4 コスト上の注意

`MERGE` はクラスタ設定の有無にかかわらずtarget/source双方を実質全走査する。クラスタリングはJOIN/フィルタの高速化には寄与するが、`MERGE` 自体のスキャン量は減らさない。**減らせるのはパーティションプルーニング（②）だけ**。

## 5. 変換基盤A：Dataform

### 5.1 ディレクトリ

```text
definitions/
├── sources/       bronze の declaration（参照定義）
├── silver/{domain}/{table}.sqlx
├── gold/{consumer}/{table}.sqlx
└── assertions/    データ品質チェック
includes/          共通JS（監査項目マクロなど）
workflow_settings.yaml
```

* **`.sqlx` はすべて `definitions/` 配下に置く**。配下でないファイルはコンパイルされず、**テストが素通りする事故**になる。
* ファイルパスは `definitions/{layer}/{option_name}/{table_name}.sqlx`。`config` の `schema` / `name` を `11` 章の命名と完全一致させる。

### 5.2 config ブロック

```js
config {
  type: "incremental",
  schema: "silver_common",          // 11章のデータセット名と一致
  name: "d_account",                // 11章のテーブル名と一致
  uniqueKey: ["account_id"],
  tags: ["silver", "silver_common"],// 実行対象の絞り込み単位
  description: "取引先マスタ（Salesforce Account 由来）",
  bigquery: { clusterBy: ["account_id"] },
  assertions: { uniqueKey: ["account_id"], nonNull: ["account_id"] }
}
```

* ファクトは `bigquery: { partitionBy: "business_date" | "DATE_TRUNC(business_date, MONTH)", clusterBy: [...] }`。10年以上保持は後者。`clusterBy` の先頭に基準日列を置く。
* `tags` は `{layer}` と `{layer}_{option_name}` の2つを必ず付ける（層一括実行とドメイン単位実行の両方を可能にするため）。

### 5.3 監査項目とタイムスタンプ

タイムスタンプとバッチIDは**コンパイル変数（`vars`）として実行のたびに外部注入**する。`.sqlx` 内で `CURRENT_TIMESTAMP()` を直書きしない。

```js
// includes/audit_columns.js
function auditColumns(sourceSystem) {
  return `
    TIMESTAMP('${dataform.projectConfig.vars.etl_ts}') AS etl_updated_at,
    '${sourceSystem}' AS etl_source_system,
    '${dataform.projectConfig.vars.batch_id}' AS etl_batch_id
  `;
}
module.exports = { auditColumns };
```

* `vars` は**コンパイル時に確定**する。Dataform APIでは `compilationResults.create` の `codeCompilationConfig.vars` で渡し、その結果を `workflowInvocations.create` で実行する。`workflowInvocations` に直接 `vars` は渡せない。
* このため **Dataform内蔵スケジュール（Workflow Config）ではなく、Workflows経由の起動を標準**とする（`22` 章）。

### 5.4 増分と範囲条件

* 範囲条件（②）は `updatePartitionFilter` に書く。Dataformがこれを `MERGE ... ON` に差し込む。
* **ソース側（①）は自分で書く**。`updatePartitionFilter` はターゲット側にしか効かない。
* ソース側の条件は **`${when(incremental(), ...)}` で囲う**。囲わないと `--full-refresh` がウィンドウ内しか作り直さなくなり、「フルリフレッシュで復旧する」運用手順が機能しなくなる。
* 削除検知（③）は `post_operations` の `UPDATE ... WHERE` で行う。`WHERE` にパーティション範囲条件を必ず含める。

### 5.5 `etl_loaded_at` の引き継ぎ

Dataformの `incremental` は `MERGE` で**全列を更新する**ため、そのままでは `etl_loaded_at`（初回挿入日時）が毎回上書きされる。既存値を引き継ぐには、`incremental()` のときだけ自テーブルへ `LEFT JOIN` する。

```sql
SELECT
  ...,
  ${when(incremental(),
    `COALESCE(prev.etl_loaded_at, ${loadedAt()})`,
    `${loadedAt()}`)} AS etl_loaded_at,
  ${auditColumns("user_api")}
FROM ${ref("bronze_user_api", "user")} AS src
${when(incremental(), `LEFT JOIN ${self()} AS prev ON CAST(src.id AS STRING) = prev.account_id`)}
```

ディメンションはクラスタ列（結合キー）で引けるため、この追加スキャンのコストは小さい。ファクトでは範囲条件を `JOIN` 側にも掛けてパーティションを絞ること。

## 6. 変換基盤B：dbt

### 6.1 ディレクトリ

```text
models/
├── sources.yml                    bronze の source 定義
├── silver/{domain}/{table}.sql
└── gold/{consumer}/{table}.sql
macros/audit_columns.sql
tests/                             汎用テスト
```

* ファイル名 = モデル名 = BigQueryのテーブル名。
* `dbt_project.yml` の `models:` で層ごとに `+schema` / `+tags` / `+materialized` を定義し、モデル個別のconfigは差分のみ書く。

### 6.2 config

```sql
{{ config(
    materialized = 'incremental',
    incremental_strategy = 'merge',
    unique_key = ['account_id'],
    cluster_by = ['account_id'],
    tags = ['silver', 'silver_common'],
    on_schema_change = 'append_new_columns'
) }}
```

* ファクトは `partition_by = {'field': 'business_date', 'data_type': 'date', 'granularity': 'month'}` と `partitions` による範囲指定を併用する。
* `incremental_strategy='insert_overwrite'` は**パーティション単位の洗替（P2）にのみ**使用してよい。

### 6.3 監査項目とタイムスタンプ

* dbtの `run_started_at` は実行単位で固定される値なので、これを使う。
* `etl_batch_id` は `var('run_id')`（Cloud Run Jobs の環境変数から `--vars` で渡す）を使う。
* 共通マクロ `{{ audit_columns('salesforce') }}` で一括付与し、各モデルに手書きしない。

### 6.4 削除検知

* dbtの `merge` は `WHEN NOT MATCHED BY SOURCE` を生成しないため、削除検知は `post_hook` の `UPDATE ... WHERE` で行う。
* `post_hook` にも**同じ範囲条件を必ず書く**。

## 7. テスト・アサーション

| 種別 | Dataform | dbt | 内容 |
| :--- | :--- | :--- | :--- |
| 一意性 | `assertions.uniqueKey` | `unique` / `dbt_utils.unique_combination_of_columns` | 主キーの重複 |
| NOT NULL | `assertions.nonNull` | `not_null` | 必須列の欠損 |
| 参照整合性 | カスタムassertion | `relationships` | ファクトのキーがディメンションに存在するか |
| 値の範囲 | カスタムassertion | `accepted_values` / `dbt_expectations` | コード値・数値の妥当性 |
| 業務ルール | カスタムassertion | singular test | 重複コマ、収支の異常値、ウィンドウ外更新の検知 |

* **主キー一意性とNOT NULLは全SILVERテーブルに必須**。無いモデルはレビューを通さない。
* テストの失敗は**後続を止める**（`22` 章）。警告扱いにしてよいのは、閾値ベースの品質指標のみ。
