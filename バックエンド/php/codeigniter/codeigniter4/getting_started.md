# はじめに（CodeIgniter 4）

## ひとことで言うと
CodeIgniter 4 を**動く状態まで持っていく初期セットアップ**のこと。プロジェクト作成 → ローカルサーバ起動 → `.env` 設定 → DB マイグレーションまでが一連の流れ。

## 役割・なぜ必要か
- CI4 は CI3 から全面刷新され、**Composer / 名前空間 / `spark` CLI** が前提。最初に「正しい雛形」を入れておかないと後のすべてがズレる。
- `.env`・`writable/` 権限・`app.baseURL` の 3 点は、設定ミスがそのまま「真っ白画面」「500」「リンク崩れ」に直結するため、最初に押さえる価値が高い。
- ローカルで `php spark serve` が起動し、`spark migrate` が通る状態 = 「開発を始められる土台」が完成したサイン。

## 基本の書き方（コード）
```bash
# 1) プロジェクト作成（推奨：appstarter テンプレート）
composer create-project codeigniter4/appstarter my-app
cd my-app

# （手動DLの場合）公式zipを展開し composer install で依存解決
#   https://codeigniter.com/download → 展開 → composer install

# 2) 環境ファイルを用意（雛形 env をコピーして .env に）
cp env .env

# 3) ローカル開発サーバ起動（既定 http://localhost:8080）
php spark serve
#   ポート変更： php spark serve --port 8081

# 4) writable/ に書き込み権限（ログ・キャッシュ・セッション保存先）
chmod -R 770 writable/   # 環境に応じて Web サーバ実行ユーザへ

# 5) DB マイグレーション実行（テーブル作成）
php spark migrate
#   ロールバック： php spark migrate:rollback
#   状態確認：     php spark migrate:status
```

```bash
# .env の主な設定（# を外して有効化する）
CI_ENVIRONMENT = development      # development / testing / production

app.baseURL = 'http://localhost:8080/'   # 末尾スラッシュ必須

database.default.hostname = localhost
database.default.database = my_db
database.default.username = root
database.default.password = secret
database.default.DBDriver  = MySQLi
```

```
# app/ 配下のおおまかな構成
app/
├── Config/        … 設定クラス（Routes.php / App.php / Database.php ...）
├── Controllers/   … コントローラ（BaseController を継承）
├── Models/        … モデル（CI4 Model）
├── Views/         … ビュー（.php テンプレート）
├── Filters/       … before/after の共通処理
├── Database/
│   ├── Migrations/  … マイグレーション
│   └── Seeds/       … シーダ
└── Helpers/       … 自作ヘルパー
public/            … 公開ディレクトリ（index.php / .htaccess）※ドキュメントルートはここ
writable/          … ログ・キャッシュ・セッション・アップロード（要書き込み権限）
```

## 実務での使い方・定番パターン
- **公開ディレクトリは必ず `public/`** に向ける。`app/` や `writable/` を Web から見せると情報漏えいになる。本番は VirtualHost の DocumentRoot を `public/` に。
- `.env` は **環境ごとに別物**。Git にはコミットしない（雛形の `env` だけ追跡）。本番は `CI_ENVIRONMENT = production` にしてエラー詳細を隠す。
- 設定値は `.env` の `名前空間.キー = 値`（例 `app.baseURL`）で **Config クラスのプロパティを上書き**できる。→ [config_env.md](./config_env.md)
- マイグレーションは `php spark make:migration CreateUsers` で雛形生成 → `up()/down()` を書く → `migrate`。チーム開発では DB を直接いじらず必ずマイグレーション経由に。
- 動作確認の最短ルート：`php spark serve` → ブラウザでトップ表示 → `php spark routes` でルート一覧確認。→ [routing.md](./routing.md)

## ハマりどころ / アンチパターン
- **`.env` を作っていない / `cp env .env` 忘れ**：設定が反映されず DB 接続エラー。`env`（ドットなし雛形）と `.env`（実体）を取り違えない。
- **`CI_ENVIRONMENT` が production のままローカル開発**：エラー画面が「白いページ」になり原因が見えない。開発中は `development` に。
- **`writable/` の権限不足**：ログ・キャッシュが書けず 500 や「真っ白」。`writable/` 全体（logs/cache/session/uploads）に書き込み権限を。
- **`app.baseURL` 未設定 / 末尾スラッシュ無し**：`base_url()` や CSS/JS リンク、リダイレクトが崩れる。`'http://localhost:8080/'` のように **末尾 `/` 必須**。
- **DocumentRoot をプロジェクト直下にする**：`public/` を経由しないと `.env` 等が丸見え。必ず `public/` を公開点に。
- `composer install` 忘れ（手動DL時）：`vendor/` が無く `autoload` で落ちる。

## フォルダ構成（始動直後）
```
myapp/
├── app/
│   ├── Controllers/
│   │   ├── BaseController.php            # 基底【生成】
│   │   ├── Home.php                      # サンプル【生成】
│   │   └── UserController.php            # 自分で作る
│   ├── Models/
│   │   └── UserModel.php                 # 自分で作る
│   ├── Views/
│   │   ├── welcome_message.php           # サンプル【生成】
│   │   ├── errors/                       # エラーページ【生成】
│   │   └── users/index.php               # 自分で作る
│   ├── Config/                           # 設定群【生成】
│   │   └── App.php  Routes.php  Database.php  Filters.php …
│   ├── Filters/                          # 前後処理（CSRF等）【生成一部】
│   ├── Database/
│   │   ├── Migrations/                   # マイグレーション（自分）
│   │   └── Seeds/                        # シード（自分）
│   ├── Entities/                         # エンティティ（自分・任意）
│   └── Helpers/  Libraries/  Language/   # 補助【生成 + 自分】
├── public/
│   ├── index.php                         # 公開エントリ【生成】
│   └── .htaccess  robots.txt  favicon.ico
├── writable/                             # 書込領域【生成】
│   └── cache/  logs/  session/  uploads/  debugbar/
├── tests/
│   └── unit/  _support/                  # テスト【生成】
├── spark                                 # CLI（migrate/make …）【生成】
├── composer.json                         # 依存（framework本体は vendor/）【生成】
├── env                                   # → .env にコピーして使う【生成】
└── phpunit.xml.dist                      # 【生成】
```
- 設定は **`app/Config/`**（`Routes.php` でルーティング、`Database.php` でDB）。公開は `public/` のみ。
- **【生成】= `composer create-project codeigniter4/appstarter`。** `Controllers`/`Models`/`Views/users` 等の中身は自分で作る。
- `writable/` はログ・キャッシュ・アップロードの書込先（権限に注意）。

## 関連
[routing.md](./routing.md) / [config_env.md](./config_env.md)
