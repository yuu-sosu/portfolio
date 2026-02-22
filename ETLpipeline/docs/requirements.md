# 要件定義書（MVP）: Ledger CSV → DB → API

## 1. 目的（Why）
- 家計簿データ（支出/収入）をCSVから取り込み、DBに格納し、APIで取得できる「最低限の作品」を作る
- 第三者がローカルで再現できる状態（docker compose + README）にする

## 2. 対象範囲（Scope）
### 対象（やる）
- 入力CSVの読み込み
- DB（PostgreSQL）への格納
- 重複しない取り込み（再実行耐性の最低限：UNIQUE + UPSERT）
- Read-only APIでの取得（期間指定）
- docker composeでの再現環境
- READMEに最短手順（起動→ETL→API）

### 対象外（やらない）
- データ品質チェックの充実（reject隔離、詳細バリデーション等）
- 増分処理の厳密設計（watermark）
- CI（lint/test）
- 認証/認可、レート制限
- UI、監視、スケジューラ、本番冗長化

## 3. 入力データ（Input）
- 形式：CSV（ローカルファイル）
- ファイル：`data/ledger.csv`
- 1レコードの粒度：1取引（支出/収入）
- 想定件数：数十〜数千行（MVPでは少量でOK）

### CSVカラム（MVP）
- `date`：YYYY-MM-DD
- `type`：`expense` / `income`
- `amount`：整数（正で統一）
- `category`：文字列
- `memo`：文字列（任意）

## 4. データ設計（DB）
### テーブル（MVP）
- `ledger`

### カラム（例）
- `source_id`（TEXT, UNIQUE, NOT NULL）※ETLで生成
- `date`（DATE, NOT NULL）
- `type`（TEXT, NOT NULL）
- `amount`（INTEGER, NOT NULL）
- `category`（TEXT, NOT NULL）
- `memo`（TEXT, NULL）
- `created_at`（TIMESTAMP, NOT NULL）
- `updated_at`（TIMESTAMP, NOT NULL）

### 一意性（重複定義）
- `source_id` を一意キーとし、同じ取引は重複しない
- `source_id` はETLで以下から生成する：
  - `hash(date,type,amount,category,memo)`

## 5. ETL要件（MVP）
- 実行：コマンド1つで実行できる（例：`python -m etl`）
- 処理：
  - CSV読込 → `source_id` 生成 → DBへUPSERT
- 再実行耐性：
  - 同じCSVを2回流しても重複が発生しない（UNIQUE + UPSERT）

## 6. API要件（MVP / Read-only）
- GET `/health`：稼働確認
- GET `/records?start=YYYY-MM-DD&end=YYYY-MM-DD`：期間指定で取引一覧を返す
  - （任意）`limit` を追加してもよい

### 返却形式
- JSON
- ソート順：date昇順または降順（どちらかに固定）

## 7. 再現性（MVP）
- `docker compose up` でDB + APIが起動する
- 初期テーブル作成はcompose起動時に自動実行される（init.sql等）
- READMEに以下が記載されている：
  - 起動手順
  - ETL実行手順
  - API確認（curl例）

## 8. 完了条件（Done）
- docker composeで起動できる
- ETLを1回実行するとDBにデータが入る
- APIで期間指定してデータが返る
- 同じCSVを2回ETLしても重複しない
- READMEの手順で第三者が再現できる