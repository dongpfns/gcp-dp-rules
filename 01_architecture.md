# 01. アーキテクチャ

## 1. 全体像

```text
[外部ソース]                [取込]                  [蓄積・変換]              [公開]
Salesforce ────┐
Box            ├─► Cloud Functions ──► GCS ──► BigQuery BRONZE ──► SILVER ──► GOLD ──► BIツール
外部API        │    Cloud Run Jobs     (raw)      (生データ)   (標準化)  (マート)   BigQuery BI Engine
CSV/ファイル ──┘    BigQuery DTS                                                        外部システム
                         ▲                              ▲
                         │                              │
                    Cloud Workflows ◄── Cloud Scheduler │
                         │                              │
                         └──── Dataform / dbt ──────────┘

[横断]  Terraform（IaC） / GitHub Actions（CI/CD） / IAM / Secret Manager / Cloud Logging / Cloud Monitoring
```

## 2. レイヤー設計（3層レイクハウス）

| 層 | 目的 | 入れてよいもの | 入れてはいけないもの |
| :--- | :--- | :--- | :--- |
| **BRONZE（RAW）** | ソースの生データを**そのまま**保持し、再構築の起点にする | ソースの列名・型のまま、取込メタ列 | 型変換・名寄せ・集計・列の削除 |
| **SILVER（DWH）** | 表記ゆれ・型・タイムゾーンを標準化し、全社で共通利用できる形にする | 行レベルの機械的変換、監査項目、論理削除フラグ | `GROUP BY`／ウィンドウ関数を伴う集約、業務KPIロジック |
| **GOLD（MART）** | 特定の利用者（BIツール・業務・システム）向けに結合・集計を済ませる | 派生指標（`kpi_` / `_rate` / `_yoy` / `_diff`）、非正規化 | 生データの直接参照（必ずSILVER経由） |

**設計原則**

1. **1方向**: BRONZE → SILVER → GOLD の一方向のみ。GOLDからSILVERを更新する処理は禁止。
2. **BRONZEは削除しない**: GCS上の原本とBRONZEテーブルは、SILVER以降を作り直すための唯一の再現手段。容量対策は削除ではなくストレージクラス移行で行う（`13` 章）。
3. **ロジックの単一実装**: 同じ業務ロジックをSILVERとGOLDの両方に書かない。派生指標はGOLDのみ（`11` 章 4.3）。
4. **層をまたぐ飛び越し禁止**: BIツールがBRONZEを直接参照することを禁止する。IAMでも実際に遮断する（`30` 章）。
5. **BRONZEに人間の命名規則を強制しない**: 連携ツールが自動生成するテーブル名・列名を無理に矯正するとパイプラインが壊れる。命名の担保はデータセット名側で行う。

## 3. 構成パターンの選定

SILVER/GOLDの変換ロジックをどこで実行するかで2案。**それ以外（取込・オーケストレーション・IaC・CI/CD）は共通**。

| | **A: Dataform**（既定） | **B: dbt on Cloud Run Jobs** |
| :--- | :--- | :--- |
| 実行基盤 | Dataform（BigQueryネイティブのマネージド） | Cloud Run Jobs 上でdbtをバッチ実行 |
| 追加で必要な基盤 | なし | Artifact Registry / Cloud Build / コンテナ運用 |
| リポジトリ | インフラ用 + Dataform専用（2リポジトリ必須） | モノレポ1本で完結可 |
| テスト | assertions（シンプル） | dbt tests + パッケージ（`dbt-utils` 等） |
| 依存グラフ | Dataform UIで可視化（追加構築不要） | `dbt docs` を生成しGCSで公開する仕組みが別途必要 |
| 学習コスト | 低（SQLXのconfigブロックのみ） | 中（Jinja / macro / profiles） |
| 向くケース | GCP内で完結させ運用工数を抑えたい | dbtの資産・知見がある、複雑なマクロ/テストを書きたい |

**判断基準（この順で確認する）**

1. 既にdbt資産・運用体制があるか → あれば B
2. マルチクラウド／BigQuery以外のDWHへ将来展開する可能性があるか → あれば B
3. 上記いずれもNo → **A（Dataform）**。運用対象コンポーネントが最も少ない。

> Dataformは「デプロイ」工程が存在せず、GitHubリポジトリをDataformが直接コンパイルする。CI/CDはデプロイではなく**検証（コンパイルとassertionのdry-run）**が役割になる（`50` 章）。

## 4. サービス選定の指針

| やりたいこと | 標準の選択 | 使わない選択とその理由 |
| :--- | :--- | :--- |
| SaaS/DBからの定期取込 | BigQuery Data Transfer Service（対応ソースがある場合） | 自前実装は保守対象が増える |
| 対応ソースがないAPIからの取込 | Cloud Functions (Gen2, Python) | 実行9分超・大量メモリなら Cloud Run Jobs |
| 実行時間が長い／重い取込 | Cloud Run Jobs | Functionsはタイムアウト上限に張り付く |
| 処理の順序制御・分岐・リトライ | Cloud Workflows | Functionsから次のFunctionsを直接呼ぶ連鎖は禁止（依存が不可視になる） |
| 定時起動 | Cloud Scheduler → Workflows | Schedulerから個別ジョブを直接叩かない（順序が表現できない） |
| ファイル到着起動 | Eventarc（GCS） → Workflows | — |
| SILVER/GOLD変換 | Dataform または dbt | Functions内でSQLを組み立てる実装は禁止（リネージが追えない） |
| 大量データの一括変換 | BigQueryのSQL（`MERGE` / `CREATE OR REPLACE`） | Pythonでの行単位処理は禁止 |

## 5. 環境

| 環境 | 環境コード | GCPプロジェクトID | 用途 |
| :--- | :--- | :--- | :--- |
| 本番 | `prd` | `{system}-prd` | 本番運用 |
| 検証 | `stg` | `{system}-stg` | 本番同等構成での受入・性能検証 |
| 開発 | `dev` | `{system}-dev` | 開発・単体検証 |

* **環境はGCPプロジェクトで分離する**。1プロジェクト内でデータセット名により環境を分ける構成は、IAM境界とコスト按分が破綻するため禁止。
* 環境間でTerraformコード（`.tf`）は共通とし、差分は `environments/{env}.tfvars` の変数値のみで表現する（`40` 章）。
