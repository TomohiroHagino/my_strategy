# プロジェクトの始め方（Rails 6）

## ひとことで言うと
`gem install rails -v "~> 6.1"` で Rails 6.1 を入れ、`rails new` でアプリ雛形を作り、DB作成・マイグレーション後に `rails server` で起動する。Rails 6 は Webpacker が標準なので Node/yarn が必須。

## 役割・なぜ必要か
Rails 6 のアプリは「Rubyのバージョン」「Rails本体」「JSビルド（Webpacker）」「DB」の4つが揃って初めて動く。Rails 7 のような `bin/dev`（foreman）は無く、サーバとWebpackビルドを別プロセスで動かす。

## 基本の書き方（コード）
```bash
ruby -v              # 2.6.x または 2.7.x（Rails 6 の実務想定）
gem install rails -v "~> 6.1"
rails -v             # Rails 6.1.x

# PostgreSQL利用・test::unit除外（-T）でアプリ生成。既定でWebpackerが入る
rails new myapp -d postgresql -T
# JS不要のAPIモード（Webpacker無し）
rails new myapp_api -d postgresql -T --api

cd myapp
bin/setup            # bundle install + yarn install + db:prepare をまとめて実行
yarn install         # Webpackerの依存JS（package.json/yarn.lock）を導入

bin/rails db:create
bin/rails db:migrate
bin/rails db:seed    # db/seeds.rb を実行

bin/rails server     # http://localhost:3000（別ターミナルで）
bin/webpack-dev-server  # JSを変更しながら開発するとき併用（自動再ビルド）

# scaffoldでCRUD一式生成（モデル/コントローラ/ビュー/ルート/マイグレーション）
bin/rails g scaffold Post title:string body:text
bin/rails db:migrate
```

## 実務での使い方・定番パターン
- 生成される主なディレクトリ：`app/controllers` `app/models` `app/views` `app/javascript/packs/application.js`（Webpackerのエントリ） `config/routes.rb` `db/migrate` `config/credentials.yml.enc`。
- JSパッケージは `bin/yarn add <pkg>`、Rubyは `bundle add <gem>` で追加する。
- 本番ビルドは `bin/rails assets:precompile`（内部で `webpack --mode production` も走る）。
- `--api` のときはビューとWebpackerが省かれ、`ActionController::API` ベースになる。

## ハマりどころ / アンチパターン
- `config/master.key` が無いと credentials を復号できず起動失敗する。チームでは共有するか `RAILS_MASTER_KEY` で渡す（[config_credentials.md](./config_credentials.md)）。
- Node/yarn 未インストールだと `rails new`（Webpacker導入）や `assets:precompile` が失敗する。`--api` 以外では必須。
- ローカルの ruby 版が Gemfile の `ruby "2.7.x"` とズレると `bundle` が止まる。`rbenv local` 等で合わせる。
- `bin/dev` は Rails 6 に存在しない。`rails server` と `bin/webpack-dev-server` を別々に起動する。

## フォルダ構成（始動直後）
```
myapp/
├── app/                                   # MVCの本体
│   ├── controllers/
│   │   ├── application_controller.rb      # 全コントローラの親
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
│   ├── channels/
│   │   └── application_cable/             # Action Cable（WebSocket）基底
│   ├── javascript/                        # JS（Webpacker）
│   │   ├── packs/
│   │   │   └── application.js             # Webpackのエントリ
│   │   └── channels/                      # Action Cable のJS購読
│   └── assets/
│       ├── stylesheets/
│       │   └── application.css            # CSSエントリ（Sprockets併存）
│       └── images/                        # 画像
├── config/
│   ├── routes.rb                          # ルーティング定義
│   ├── application.rb                     # アプリ全体設定
│   ├── database.yml                       # DB接続設定（複数DB可）
│   ├── webpacker.yml                      # Webpacker のビルド設定
│   ├── cable.yml                          # Action Cable 設定
│   ├── puma.rb                            # Webサーバ設定
│   ├── credentials.yml.enc               # 暗号化された秘密情報（環境別も可）
│   ├── master.key                         # 復号鍵（gitignore対象）
│   ├── environments/
│   │   ├── development.rb                 # 開発環境設定
│   │   ├── test.rb                        # テスト環境設定
│   │   └── production.rb                  # 本番環境設定
│   └── initializers/                      # 起動時に読む設定群
├── db/
│   ├── migrate/                           # マイグレーションファイル
│   ├── schema.rb                          # 現在のスキーマ（自動生成）
│   └── seeds.rb                           # 初期データ投入スクリプト
├── lib/
│   └── tasks/                             # 自作rakeタスク
├── bin/                                   # rails / setup などの実行スクリプト
├── public/                                # 静的ファイル公開ディレクトリ
├── test/                                  # テスト（RSpecなら spec/）
├── Gemfile                                # 依存gem定義
├── Gemfile.lock                           # 依存の固定版
├── package.json                           # JS依存（Webpacker/yarn）
├── babel.config.js                        # Babel設定（Webpacker）
├── postcss.config.js                      # PostCSS設定（Webpacker）
├── Rakefile                               # rakeのエントリ
└── config.ru                             # Rack起動入口
```
- JSは Webpacker（app/javascript/packs）＋ rails-ujs ＋ Turbolinks 5。
- Zeitwerk 前提なのでファイル名と定数名を一致させる。

## 関連
[routing.md](./routing.md) / [active_record.md](./active_record.md) / [config_credentials.md](./config_credentials.md) / [webpacker.md](./webpacker.md) / [console.md](./console.md) / [zeitwerk.md](./zeitwerk.md)
