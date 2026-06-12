# プロジェクトの始め方（Rails 5）

## ひとことで言うと
`rails new` で雛形を生成し、DBを作り、サーバを起動するまでの **「ゼロから動くアプリ」までの初期セットアップ手順**。Rails 5 では `--api`（JSONのみ）も選べる。

## 役割・なぜ必要か
- Rails は規約で大量のファイル/設定を自動生成する。最初の `rails new` の選択（DB・APIか等）が後々の構成を決めるので、ここを押さえると立ち上げで迷わない。
- Rails 5 では `rails new` で生成される基底クラス（`ApplicationRecord` 等）やフロント構成（Turbolinks + rails-ujs）が 6/7 と異なるため、生成物の前提を理解しておく。

## 0. 前提（入っているもの）
```bash
ruby -v        # Rails 5.0/5.1 は Ruby 2.2.2+、5.2 は 2.2.2+（実務は 2.4〜2.6 が多い）
gem install rails -v "~> 5.2"
rails -v
# JSをWebpackerで扱う場合（5.1〜）だけ node / yarn が要る。Sprockets既定ならNode不要。
```
- バージョン管理は rbenv などで固定（`.ruby-version` を置く）。Rails 5 は古いので Ruby も古めに合わせる必要がある。

## 1. アプリ生成（`rails new`）
```bash
# 標準（SQLite）
rails new myapp

# 実務でよく使う形：PostgreSQL ＋ RSpec前提でテスト雛形を省く
rails new myapp -d postgresql -T

# API専用（ビュー無し／JSONのみ）→ Rails 5 で追加されたモード
rails new myapp --api -d postgresql -T

# Webpacker（JSをyarnで管理）を使う（5.1〜）
rails new myapp --webpack
```
主なオプション:
- `-d postgresql|mysql|sqlite3` … DB
- `--api` … ビュー無しのAPIモード（→ [api_mode.md](./api_mode.md)）
- `--webpack[=react|vue|...]` … Webpacker を導入（5.1〜。yarn 必須）
- `-T` … test/ 雛形をスキップ（RSpecを入れる場合）
- `--skip-action-cable` / `--skip-active-storage`（5.2）など … 不要機能を外す

## 2. 依存と初期化
```bash
cd myapp
bundle install
yarn install          # Webpacker を使う場合のみ（5.1〜）
```
- Rails 5.0 には `bin/setup` 雛形があるが内容は薄い。基本は `bundle install` ＋ DB準備を手で行う。

## 3. データベース
```bash
# config/database.yml を確認（PostgreSQLなら接続情報 / ENV）
rails db:create        # DB作成（Rails 5 から rake でなく rails でOK）
rails db:migrate       # マイグレーション適用
rails db:seed          # 初期データ（db/seeds.rb）
```
- **Rails 5 のポイント**：`rake db:migrate` でも動くが、`rails db:migrate` が推奨に統一された。
→ 詳細は [active_record.md](./active_record.md) / [config_credentials.md](./config_credentials.md)

## 4. サーバ起動
```bash
rails server           # http://localhost:3000（Puma が既定の開発サーバ）
# Webpacker利用時、開発でJSを自動ビルドするなら別タブで:
./bin/webpack-dev-server
```

## 5. 最初の機能を作る（generator / scaffold）
```bash
# CRUD一式（モデル＋コントローラ＋ビュー＋ルート＋マイグレーション）を生成
rails generate scaffold Post title:string body:text
rails db:migrate
# 個別に
rails g model Comment post:references body:text
rails g controller Pages about
```
生成物の対応 → [routing.md](./routing.md) / [controller.md](./controller.md) / [model.md](./model.md) / [view.md](./view.md)

## 生成されるディレクトリ（ざっくり）
```
app/        # MVCの本体（models / controllers / views / helpers / jobs / mailers / channels）
config/     # routes.rb / database.yml / secrets.yml（5.2はcredentials） / environments / initializers
db/         # migrate / schema.rb / seeds.rb
lib/  test/(or spec/)  public/  bin/  Gemfile
# Webpacker導入時のみ app/javascript/ と config/webpacker.yml が増える（5.1〜）
```

## 開発の基本ループ
1. ルート定義（`config/routes.rb`）→ [routing.md](./routing.md)
2. モデル＆マイグレーション → [model.md](./model.md) / [active_record.md](./active_record.md)
3. コントローラ＆ビュー → [controller.md](./controller.md) / [view.md](./view.md)
4. 動作確認（`rails console` / ブラウザ）→ [console.md](./console.md)
5. テスト → [testing.md](./testing.md)

## ハマりどころ
- **Rubyバージョンが新しすぎる**：Rails 5 は新しいRuby（3.x）では動かない。Ruby 2.4〜2.6 あたりに合わせる必要がある。
- **DB接続エラー**：PostgreSQL/MySQL が起動していない・`database.yml` のユーザ/パスワード不一致。`db:create` 前にDBサーバを立てる。
- **`belongs_to` 必須化で scaffold 後に保存できない**：`post:references` で生成した子モデルは親が未設定だと検証エラー。任意にするなら `optional: true`。→ [pitfalls.md](./pitfalls.md)
- **yarn が無い**：`--webpack` を付けたのに yarn 未インストールでアセットが動かない（5.1〜）。Sprockets だけなら不要。
- `-T` で生成したのに test/ を探す（RSpec なら spec/）。

## フォルダ構成（始動直後）
```
myapp/
├── app/                                   # MVCの本体
│   ├── controllers/
│   │   └── application_controller.rb      # 全コントローラの親
│   ├── models/
│   │   └── application_record.rb          # 全モデルの親（ActiveRecord::Base継承）
│   ├── views/
│   │   └── layouts/                       # 共通レイアウト
│   ├── helpers/                           # ビュー用ヘルパー
│   ├── jobs/
│   │   └── application_job.rb             # ActiveJob の親
│   ├── mailers/
│   │   └── application_mailer.rb          # ActionMailer の親
│   ├── channels/
│   │   └── application_cable/             # Action Cable（WebSocket）基底
│   └── assets/                            # Sprockets
│       ├── javascripts/
│       │   └── application.js             # JSエントリ（rails-ujs＋Turbolinks）
│       ├── stylesheets/
│       │   └── application.css            # CSSエントリ
│       └── images/                        # 画像
├── config/
│   ├── routes.rb                          # ルーティング定義
│   ├── application.rb                     # アプリ全体設定
│   ├── database.yml                       # DB接続設定
│   ├── cable.yml                          # Action Cable 設定
│   ├── puma.rb                            # Webサーバ設定
│   ├── secrets.yml                        # 秘密情報【5.0/5.1】
│   ├── credentials.yml.enc               # 暗号化された秘密情報【5.2】
│   ├── master.key                         # 復号鍵【5.2】（gitignore対象）
│   ├── environments/                      # development/test/production 設定
│   └── initializers/                      # 起動時に読む設定群
├── db/
│   ├── migrate/                           # マイグレーションファイル
│   ├── schema.rb                          # 現在のスキーマ（自動生成）
│   └── seeds.rb                           # 初期データ投入スクリプト
├── lib/                                   # 自作ライブラリ・rakeタスク
├── bin/                                   # rails / setup などの実行スクリプト
├── public/                                # 静的ファイル公開ディレクトリ
├── test/                                  # テスト（RSpecなら spec/）
├── Gemfile                                # 依存gem定義
└── config.ru                             # Rack起動入口
```
（5.1〜 Webpacker選択時のみ app/javascript/ と config/webpacker.yml が増える）
- JSは rails-ujs ＋ Turbolinks 5、アセットは Sprockets（app/assets）。
- オートロードは classic（5.0/5.1）。`rails new --api` だと views/assets が省かれ API 構成になる。

## 関連
[routing.md](./routing.md) / [active_record.md](./active_record.md) / [api_mode.md](./api_mode.md) / [config_credentials.md](./config_credentials.md) / [console.md](./console.md)
