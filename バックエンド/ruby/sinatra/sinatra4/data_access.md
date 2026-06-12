# DB / データアクセス（Data Access）（Sinatra 3/4）

## ひとことで言うと
**Sinatraには“DB層”が無い**、という話。SinatraはORMもDB接続も**内蔵しない**ので、データを永続化したければ `sinatra-activerecord`（ActiveRecord）や `Sequel` を**自分でgemとして足し**、接続・マイグレーションも自前で組む。

## 役割・なぜ必要か
- Railsは `ActiveRecord` ・マイグレーション・コネクション管理が**最初から全部入っている**。Sinatraは真逆で、**素のままだとDBの仕組みは一切無い**。
- だから「モデルはどこ？」「`rails db:migrate` は？」と**Railsの感覚で探しても何も見つからない**。これは欠落ではなく設計思想（必要な物だけ足す）。
- 結果として、**データの構造設計・接続・マイグレーション・命名規約まで自分で決める**ことになる。小さいうちは身軽、大きくなると負担。

## 基本の書き方（コード）
```ruby
# Gemfile … ORMを“自分で足す”。例: ActiveRecord 系
# gem "sinatra"
# gem "sinatra-activerecord"     # SinatraにARを繋ぐ橋渡し
# gem "rake"                     # マイグレーション用タスク
# gem "sqlite3"                  # ドライバ（DBに合わせて pg 等）

require "sinatra"
require "sinatra/activerecord"   # これで DB 接続/モデルが使えるようになる

# 接続設定（自分で書く）
set :database, { adapter: "sqlite3", database: "db/development.sqlite3" }

# モデルも自分で定義（テーブル users ↔ クラス User）
class User < ActiveRecord::Base
end

get "/users" do
  @users = User.order(created_at: :desc)  # ここで初めてARが使える
  erb :users
end

post "/users" do
  User.create!(name: params[:name])       # 例外で失敗を拾う
  redirect "/users"
end
```

```ruby
# Rakefile … マイグレーションのタスクも別gem(sinatra-activerecord/rake)で足す
require "sinatra/activerecord/rake"
require "./app"   # アプリ本体を読み込む

# 使い方:
#   bundle exec rake db:create_migration NAME=create_users
#   bundle exec rake db:migrate
```

```ruby
# 代替案: Sequel（ARより薄い・軽量なORM）。同じく自分で足す
# Gemfile: gem "sequel", gem "sqlite3"
require "sequel"
DB = Sequel.connect("sqlite://db/dev.sqlite3")   # 接続は自前
DB.create_table?(:users) { primary_key :id; String :name }  # マイグレ相当も自前
users = DB[:users]
users.insert(name: "Taro")
```

## 実務での使い方・定番パターン
- **gemを足してから初めてDBが使える**：`sinatra-activerecord`（ARを使いたい人）か `Sequel`（薄く速く）を選ぶ。どちらが正解ということはなく好みと規模次第。
- **接続設定は自分で書く**：`set :database, {...}` か `DATABASE_URL` 環境変数。本番/開発の出し分けも自前（environments と組み合わせる）。
- **マイグレーション**も別物：`sinatra-activerecord/rake` を入れて `rake db:create_migration` / `rake db:migrate`。Sequelなら `sequel` のmigration機構を使う。
- **モデルの中身（バリデーション・関連・スコープ）はActiveRecordそのもの**。書き味はRailsと同じなので、Rails側の知識を流用できる。→ [../rails/rails7/active_record.md](../../rails/rails7/active_record.md)
- 小物なら**そもそもDBを使わず**ファイル/メモリ/外部APIで済ませる選択も普通。YAGNIで「本当にDBが要るか」から判断する。

## ハマりどころ / アンチパターン
- **「何も無い」前提を忘れる（最重要）**：Sinatra単体には接続もモデルもマイグレーションも無い。`User` が動かないのは“バグ”ではなく、**gemと設定をまだ足していないだけ**。
- **Railsの感覚で探さない**：`app/models/` や `db:migrate` を探しても無い。Sinatraでは置き場も命名も**自分で構造設計する**（規約が無いので決め事を作る）。
- **接続のライフサイクルを放置**：Webサーバ（Puma等）のマルチスレッド/プロセスでコネクションプールを意識しないと枯渇・リーク。ARなら `ActiveRecord::Base.connection_pool` 周りを確認。
- **マイグレーションgemの入れ忘れ**：`sinatra-activerecord` 本体だけ入れて `rake` 連携を忘れると `rake db:migrate` が無い、で詰まりやすい。
- **ORMの混在**：ARとSequelを両方入れて中途半端に使うと接続が二重化して混乱。**どちらか一本**に決める。
- ハードコードした接続情報をコミットしない。秘密情報は環境変数へ。→ [config_testing.md](./config_testing.md)

## 関連
[../rails/rails7/active_record.md](../../rails/rails7/active_record.md) / [config_testing.md](./config_testing.md) / [rack_and_filters.md](./rack_and_filters.md)
