# プロジェクトの始め方（Rails 7）

## ひとことで言うと
`rails new` で雛形を生成し、DBを作り、サーバを起動するまでの **「ゼロから動くアプリ」までの初期セットアップ手順**。

## 役割・なぜ必要か
- Railsは規約で大量のファイル/設定を自動生成する。最初の `rails new` の選択（DB・フロント・APIか等）が後々の構成を決めるので、ここを押さえると立ち上げで迷わない。

## 0. 前提（入っているもの）
```bash
ruby -v        # Ruby 3.x（Rails 7 は Ruby 2.7+、実務は 3.1+ 推奨）
gem install rails -v "~> 7.1"
rails -v
# importmap既定ならNode不要。jsbundling/cssbundlingを使うときだけ node / yarn が要る
```
- バージョン管理は rbenv / mise などで固定（`.ruby-version` を置く）。

## 1. アプリ生成（`rails new`）
```bash
# 標準（SQLite）
rails new myapp

# 実務でよく使う形：PostgreSQL ＋ Tailwind ＋ RSpec前提でテスト雛形を省く
rails new myapp -d postgresql --css tailwind -T

# API専用（フロントは別／JSONのみ）
rails new myapp --api -d postgresql -T
```
主なオプション:
- `-d postgresql|mysql|sqlite3` … DB
- `--css tailwind|bootstrap|sass` / `--javascript importmap|esbuild|bun`
- `--api` … ビュー無しのAPIモード
- `-T` … test/ 雛形をスキップ（RSpecを入れる場合）
- `--skip-...` … 不要な機能を外す（action_mailbox 等）

## 2. 依存と初期化
```bash
cd myapp
bin/setup            # bundle install ＋ DB準備などをまとめて実行（雛形に同梱）
# 手動なら:
bundle install
```

## 3. データベース
```bash
# config/database.yml を確認（PostgreSQLなら接続情報 / ENV）
bin/rails db:create        # DB作成
bin/rails db:migrate       # マイグレーション適用
bin/rails db:seed          # 初期データ（db/seeds.rb）
```
→ 詳細は [active_record.md](./active_record.md)（マイグレーション）/ [config_credentials.md](./config_credentials.md)（接続情報・秘密情報）

## 4. サーバ起動
```bash
bin/rails server           # http://localhost:3000
# フロントのビルドも同時に回す構成なら:
bin/dev                    # Procfile.dev（web: rails s / css: tailwind --watch 等）
```

## 5. 最初の機能を作る（generator / scaffold）
```bash
# CRUD一式（モデル＋コントローラ＋ビュー＋ルート＋マイグレーション）を生成
bin/rails generate scaffold Post title:string body:text
bin/rails db:migrate
# 個別に
bin/rails g model Comment post:references body:text
bin/rails g controller Pages about
```
生成物の対応 → [routing.md](./routing.md) / [controller.md](./controller.md) / [model.md](./model.md) / [view.md](./view.md)

## 生成されるディレクトリ（ざっくり）
```
app/        # MVCの本体（models / controllers / views / helpers / jobs / mailers）
config/     # routes.rb / database.yml / credentials / environments / initializers
db/         # migrate / schema.rb / seeds.rb
lib/  test/(or spec/)  public/  bin/  Gemfile
```

## 開発の基本ループ
1. ルート定義（`config/routes.rb`）→ [routing.md](./routing.md)
2. モデル＆マイグレーション → [model.md](./model.md) / [active_record.md](./active_record.md)
3. コントローラ＆ビュー → [controller.md](./controller.md) / [view.md](./view.md)
4. 動作確認（`bin/rails console` / ブラウザ）→ [console.md](./console.md)
5. テスト → [testing.md](./testing.md)

## ハマりどころ
- **Rubyバージョン不一致**：`.ruby-version` と環境がズレてbundle失敗。rbenv/miseで合わせる。
- **DB接続エラー**：PostgreSQL/MySQLが起動していない・`database.yml`のユーザ/パスワード不一致。`db:create` 前にDBサーバを立てる。
- **`master.key` が無い**：他人のリポジトリをcloneすると `config/master.key` が無く credentials を読めない。鍵を共有してもらう（Git管理外）。→ [config_credentials.md](./config_credentials.md)
- **Node/yarn 必須と誤解**：importmap既定ならNode不要。esbuild等を選んだ時だけ必要。
- `-T` で生成したのに test/ を探す（RSpecなら spec/）。

## フォルダ構成（始動直後）
```
myapp/
├── app/                                   # MVCの本体
│   ├── controllers/
│   │   ├── application_controller.rb      # 全コントローラの親（共通処理）
│   │   └── concerns/                      # コントローラ共通モジュール
│   ├── models/
│   │   ├── application_record.rb          # 全モデルの親（ActiveRecord::Base継承）
│   │   └── concerns/                      # モデル共通モジュール
│   ├── views/
│   │   └── layouts/
│   │       └── application.html.erb       # 全画面共通レイアウト
│   ├── helpers/
│   │   └── application_helper.rb          # ビュー用ヘルパー
│   ├── jobs/
│   │   └── application_job.rb             # ActiveJob の親
│   ├── mailers/
│   │   └── application_mailer.rb          # ActionMailer の親
│   ├── javascript/                        # JS（importmap）
│   │   ├── application.js                 # JSエントリ
│   │   └── controllers/                   # Stimulusコントローラ群
│   └── assets/
│       ├── stylesheets/
│       │   └── application.css            # CSSエントリ
│       ├── config/
│       │   └── manifest.js                # Sprockets マニフェスト
│       └── images/                        # 画像
├── config/
│   ├── routes.rb                          # ルーティング定義
│   ├── application.rb                     # アプリ全体設定
│   ├── boot.rb                            # 起動初期化
│   ├── environment.rb                     # 環境読み込みの入口
│   ├── database.yml                       # DB接続設定
│   ├── importmap.rb                       # importmap のピン定義
│   ├── puma.rb                            # Webサーバ設定
│   ├── credentials.yml.enc                # 暗号化された秘密情報
│   ├── master.key                         # 復号鍵（gitignore対象）
│   ├── environments/
│   │   ├── development.rb                 # 開発環境設定
│   │   ├── test.rb                        # テスト環境設定
│   │   └── production.rb                  # 本番環境設定
│   ├── initializers/                      # 起動時に読む設定群
│   └── locales/
│       └── en.yml                         # 翻訳辞書（i18n）
├── db/
│   ├── migrate/                           # マイグレーションファイル
│   ├── schema.rb                          # 現在のスキーマ（自動生成）
│   └── seeds.rb                           # 初期データ投入スクリプト
├── lib/
│   └── tasks/                             # 自作rakeタスク
├── bin/
│   ├── rails                              # rails コマンド
│   ├── setup                              # セットアップスクリプト
│   └── dev                                # bin/dev（foreman起動）
├── public/                                # 静的ファイル公開ディレクトリ
├── test/                                  # テスト（RSpecなら spec/）
├── Gemfile                                # 依存gem定義
├── Gemfile.lock                           # 依存の固定版
├── Rakefile                               # rakeのエントリ
├── config.ru                             # Rack起動入口
└── .ruby-version                          # Rubyバージョン固定
```
- JSは importmap ＋ Hotwire（Turbo/Stimulus）。Stimulusは app/javascript/controllers に置く。Node不要。
- 秘密情報は credentials.yml.enc に暗号化保存。復号鍵 master.key は gitignore 対象（共有は別経路）。

## 関連
[routing.md](./routing.md) / [active_record.md](./active_record.md) / [config_credentials.md](./config_credentials.md) / [console.md](./console.md)
