# 配送日報Web化 サンプル｜データ設計

「現場入力できる」「Excelに戻せる」「AIに渡せる」 — この3点を満たす最小構成のデータモデルです。

## ER図（概念）

[02-er-diagram.svg](assets/02-er-diagram.svg)

主要テーブルは7つです。配送員・配送先・品目はマスタとして既存システム／Excelからの移行を想定します。

## テーブル定義

### `users`（利用者）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| code | varchar(20) UNIQUE | 社員番号など既存IDと突き合わせ可能なコード |
| name | varchar(100) | 氏名 |
| role | varchar(20) | `driver` / `office` / `admin` |
| active | boolean | 退職／休職時の論理削除 |
| created_at, updated_at | timestamp | |

### `customers`（配送先マスタ）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| code | varchar(20) UNIQUE | 既存システムの顧客コード（CSV互換のキー） |
| name | varchar(200) | 配送先名 |
| address | varchar(300) | |
| area | varchar(50) | エリア区分（ルート組みのヒント） |
| note | text | 「裏口から搬入」「14時以降不在」など申し送り |
| active | boolean | |

### `items`（品目マスタ）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| code | varchar(20) UNIQUE | |
| name | varchar(200) | |
| unit | varchar(20) | `本` / `ケース` / `kg` など |
| category | varchar(50) | 酒類／調味料／乾物 |
| active | boolean | |

### `daily_reports`（日報ヘッダ）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| report_date | date | 配送日 |
| driver_id | bigint FK→users | |
| shift | varchar(10) | `am` / `pm` / `extra` |
| depart_at | timestamp | 出発時刻 |
| return_at | timestamp | 帰庫時刻 |
| odometer_start | int | 走行距離（出発時） |
| odometer_end | int | 走行距離（帰庫時） |
| memo | text | 顧客対応メモ・全般所感 |
| has_incident | boolean | 異常・ヒヤリハットの有無（UI赤バッジ用） |
| status | varchar(10) | `draft` / `submitted` / `confirmed` |
| submitted_at | timestamp | |
| created_at, updated_at | timestamp | |

UNIQUE制約: `(report_date, driver_id, shift)` — 同日同便の重複登録を防ぐ。

### `daily_report_lines`（日報明細：配送先ごとの納品内容）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| daily_report_id | bigint FK→daily_reports | |
| seq | smallint | 訪問順 |
| customer_id | bigint FK→customers | |
| arrived_at | timestamp NULL | 到着時刻（記録があれば） |
| memo | text | 個別顧客への申し送り |
| created_at, updated_at | timestamp | |

### `daily_report_line_items`（明細の中の品目）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| daily_report_line_id | bigint FK | |
| item_id | bigint FK→items | |
| quantity | numeric(10,2) | |
| is_shortage | boolean | 積み残しフラグ |
| note | text | 「在庫切れで翌日対応」など |

### `incidents`（異常・ヒヤリハット）

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| daily_report_id | bigint FK | |
| kind | varchar(20) | `damage` / `delay` / `nearmiss` / `complaint` / `other` |
| severity | smallint | 1〜3 |
| description | text | |
| created_at | timestamp | |

### （AI第2版以降）`report_summaries`

| 列 | 型 | 説明 |
| --- | --- | --- |
| id | bigint PK | |
| scope | varchar(10) | `daily` / `weekly` / `driver` |
| target_date | date | 対象日（週次なら週初日） |
| target_user_id | bigint FK NULL | ドライバ単位の場合 |
| summary | text | AI生成の要約（編集可） |
| model | varchar(50) | 使用モデルのバージョン記録 |
| generated_at | timestamp | |

## 権限設計（最低限）

| ロール | 自分の日報 | 他人の日報 | マスタ編集 | CSV出力 | AI要約 |
| --- | --- | --- | --- | --- | --- |
| driver（配送員） | 入力／当日中の編集 | 不可 | 不可 | 不可 | 自分の分のみ閲覧 |
| office（事務） | 閲覧 | 閲覧／検索 | 不可 | 可 | 閲覧 |
| admin（管理者） | 閲覧／編集 | 閲覧／編集 | 可 | 可 | 閲覧／再生成 |

> **編集ログは `audit_logs` テーブルに残す**（日報の修正履歴は紙時代より厳格に追える）。

## CSV出力との対応

既存Excelの列順（`日付, 配送員, 便, 配送先コード, 配送先名, 品目, 数量, 異常, メモ`）を出すには、`daily_report_line_items` を中心に以下のJOINで1行ずつ展開します。

```
daily_report_line_items
  JOIN daily_report_lines      ON ...
  JOIN daily_reports          ON ...
  JOIN users (driver)         ON ...
  JOIN customers              ON ...
  JOIN items                  ON ...
  LEFT JOIN incidents (任意)   ON ...
```

文字コードは `Shift_JIS`／`UTF-8 (BOM付)` を選択可能にし、既存マクロを壊さない設計です。

## 移行データの取り込み

- 既存の月次Excelを **CSV → 一括取り込みスクリプト** で読み込み、過去半年分を初期データとして投入
- マスタ（配送先・品目）は CSV 同期コマンドを用意し、kintone・販売管理側を**マスタとして扱う**
- 「Web化を機にマスタ整備をやり直す」のではなく、**今動いているマスタをそのまま受け入れる**

## 次のドキュメント

- [03-ai-usage.md](03-ai-usage.md) — AI要約・検索・異常検知
- [04-release-plan.md](04-release-plan.md) — 段階リリース計画
