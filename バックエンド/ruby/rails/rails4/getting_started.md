# プロジェクトの始め方（Rails 4）

## ひとことで言うと
`rails new` で雛形を生成し、DBを作り、サーバを起動するまでの **「ゼロから動くアプリ」までの初期セットアップ手順**。

## 役割・なぜ必要か
- Railsは規約で大量のファイル/設定を自動生成する。最初の `rails new` の選択（DB・テストフレームワーク等）が後の構成を決めるので、ここを押さえると立ち上げで迷わない。

## 0. 前提（入っているもの）
```bash
ruby -v                 # Ruby 1.9.3 / 2.0+（4.2は2.0+。実務は2.1〜2.3が多い）
gem install rails -v "4.2.11.3"
rails -v                # Rails 4.2.11.3
node -v                 # CoffeeScript/uglifier 用にJSランタイムが要る（therubyracer でも可）
```
- バージョン管理は rbenv / rvm で固定（`.ruby-version` を置く）。
- （7では importmap で Node 不要だが、4 は Sprockets が JS ランタイムを必要とする。`therubyracer` を Gemfile に入れるか Node を入れる。）

## 1. アプリ生成（`rails new`）
```bash
# 標準（SQLite）
rails new myapp

# 実務でよく使う形：PostgreSQL ＋ test/ 雛形を省く（RSpec前提）
rails new myapp -d postgresql -T

# MySQL
rails new myapp -d mysql
```
主なオプション:
- `-d postgresql|mysql|sqlite3` … DB
- `-T` … `test/` 雛形をスキップ（RSpecを入れる場合）
- `--skip-turbolinks` … Turbolinks Classic を外す（不具合の温床なので外す現場も多い）
- `--skip-bundle` … 生成時に `bundle install` しない
- （4 には `--api` モードは無い＝Rails 5から。API専用にしたい場合も通常生成からビューを削る/`render json:` を使う。`--css` `--javascript` オプションも無い。）

## 2. 依存と初期化
```bash
cd myapp
bundle install
# bin/setup は 4.2 から雛形に入る。4.0/4.1 には無いので bundle install を直接叩く
```

## 3. データベース
```bash
# config/database.yml を確認（PostgreSQLなら接続情報 / ENV）
rake db:create        # DB作成
rake db:migrate       # マイグレーション適用
rake db:seed          # 初期データ（db/seeds.rb）
# bin/rake / bin/rails でも可。Rails 4 では rake が主流
```
→ 詳細は [active_record.md](./active_record.md)（マイグレーション）/ [config_secrets.md](./config_secrets.md)（接続情報・秘密情報）

## 4. サーバ起動
```bash
rails server           # http://localhost:3000（WEBrickが既定。Pumaが既定になるのは5系）
# 本番は unicorn 等を別途使う → ../周辺インフラ/unicorn.md
```

## 5. 最初の機能を作る（generator / scaffold）
```bash
# CRUD一式（モデル＋コントローラ＋ビュー＋ルート＋マイグレーション）を生成
rails generate scaffold Post title:string body:text
rake db:migrate
# 個別に
rails g model Comment post:references body:text
rails g controller Pages about
```
生成物の対応 → [routing.md](./routing.md) / [controller.md](./controller.md) / [model.md](./model.md) / [view.md](./view.md)

## 生成されるディレクトリ（ざっくり）
```
app/        # MVCの本体（models / controllers / views / helpers / mailers）
            #   ※ jobs / は 4.2 で active_job 導入後。4.0/4.1には無い
app/assets/ # Sprockets（javascripts / stylesheets / images）
config/     # routes.rb / database.yml / secrets.yml(4.1〜) / environments / initializers
db/         # migrate / schema.rb / seeds.rb
lib/  test/(or spec/)  public/  bin/  Gemfile
```

## 開発の基本ループ
1. ルート定義（`config/routes.rb`）→ [routing.md](./routing.md)
2. モデル＆マイグレーション → [model.md](./model.md) / [active_record.md](./active_record.md)
3. コントローラ＆ビュー → [controller.md](./controller.md) / [view.md](./view.md)
4. 動作確認（`rails console` / ブラウザ）→ [console.md](./console.md)
5. テスト → [testing.md](./testing.md)

## ハマりどころ
- **Rubyバージョン不一致**：Rails 4.2 は Ruby 2.0+。古い 1.8 系では動かない。rbenv/rvm で合わせる。
- **JSランタイムが無い**：Sprockets が CoffeeScript/uglifier の実行に JS ランタイムを要求。`ExecJS::RuntimeUnavailable` が出たら `therubyracer` を Gemfile に追加するか Node を入れる。
- **DB接続エラー**：PostgreSQL/MySQLが起動していない・`database.yml` のユーザ/パスワード不一致。`db:create` 前にDBサーバを立てる。
- **`secret_key_base` が無い**：4.1〜は `config/secrets.yml` に必要。本番は ENV から読む。→ [config_secrets.md](./config_secrets.md)
- **Turbolinks Classic の事故**：ページ遷移で自前JSが動かない/二重に動く。`--skip-turbolinks` で外すか `page:load` を使う。→ [javascript.md](./javascript.md)
- `-T` で生成したのに `test/` を探す（RSpecなら `spec/`）。

## フォルダ構成（始動直後）
```
myapp/
├── app/                                   # MVCの本体
│   ├── controllers/
│   │   └── application_controller.rb      # 全コントローラの親
│   ├── models/                            # ※ApplicationRecord無し。各モデルが
│   │                                      #   ActiveRecord::Base を直接継承
│   ├── views/
│   │   └── layouts/
│   │       └── application.html.erb       # 全画面共通レイアウト
│   ├── helpers/
│   │   └── application_helper.rb          # ビュー用ヘルパー
│   ├── mailers/                           # ActionMailer
│   └── assets/                            # Sprockets（app/javascript は無い）
│       ├── javascripts/
│       │   └── application.js(.coffee)    # JSエントリ（CoffeeScript可）
│       ├── stylesheets/
│       │   └── application.css(.scss)     # CSSエントリ（SCSS可）
│       └── images/                        # 画像
├── config/
│   ├── routes.rb                          # ルーティング定義
│   ├── application.rb                     # アプリ全体設定
│   ├── database.yml                       # DB接続設定
│   ├── secrets.yml                        # secret_key_base 等（4.1〜）
│   ├── environments/                      # development/test/production 設定
│   └── initializers/
│       └── secret_token.rb                # トークン設定（4.0系）等
├── db/
│   ├── migrate/                           # マイグレーションファイル
│   ├── schema.rb                          # 現在のスキーマ（自動生成）
│   └── seeds.rb                           # 初期データ投入スクリプト
├── lib/
│   └── tasks/                             # 自作rakeタスク
├── bin/                                   # rails / rake などの実行スクリプト
├── public/                                # 静的ファイル公開ディレクトリ
├── test/                                  # テスト（RSpecなら spec/）
├── Gemfile                                # 依存gem定義
├── Rakefile                               # rakeのエントリ
└── config.ru                             # Rack起動入口
```
- コードの主戦場は `app/`、ルーティングは `config/routes.rb`。
- app/javascript は無い。JSは jquery_ujs ＋ Turbolinks Classic、アセットは Sprockets のみ。

## 関連
[routing.md](./routing.md) / [active_record.md](./active_record.md) / [config_secrets.md](./config_secrets.md) / [console.md](./console.md)
