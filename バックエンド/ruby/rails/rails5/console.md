# rails console / コマンド（Rails 5）

## ひとことで言うと
**アプリの環境を読み込んだ状態でRubyを対話実行できるツール（`rails console`）** と、開発でよく使う **`rails` コマンド一式**。Rails 5 から多くが `rake` でなく `rails` に統一された。

## 役割・なぜ必要か
- モデルやサービスを実際に呼び出してデータを確認・操作したり、挙動を試したりするためにある（デバッグ・調査・手動修正）。
- コマンド群は生成（generator）・DB操作・サーバ起動・ルート確認など日常作業の入口。

## rails console の基本
```bash
rails console            # 開発環境で起動（rails c でも可）
rails console -e production   # 本番環境（操作は慎重に）
rails console --sandbox  # 終了時に全変更をロールバック（試し用に安全）
```
```ruby
# console 内
User.count
u = User.find(1)
u.update!(name: "新名前")
Post.where(published: true).limit(5)
reload!                  # コードを変更したら再読み込み
app.get "/posts"         # ルーティング/リクエストの簡易確認
helper.number_to_currency(1000)   # ヘルパーを試す
```

## よく使う rails コマンド（Rails 5 で rails に統一）
```bash
rails server             # サーバ起動（rails s）
rails console            # コンソール（rails c）

# 生成
rails generate model Post title:string   # rails g model ...
rails generate controller Pages about
rails generate migration AddXToPosts x:string
rails destroy model Post                 # 生成の取り消し

# DB（Rails 5 から rake でなく rails でOK）
rails db:create
rails db:migrate
rails db:rollback
rails db:seed
rails db:reset           # drop → create → schema 読み込み → seed

# 確認系
rails routes             # ルート一覧（rails routes -g post で絞り込み）
rails about              # バージョン等の情報
rails stats              # コード行数統計
rails runner "puts User.count"   # スクリプトを1回実行
```

## 実務での使い方・定番パターン
- **`--sandbox`** で本番に近いデータ操作を試す（終了時に巻き戻る）。
- **`rails runner`** で cron やワンショットのバッチを実行（`rails runner "DailyReport.new.run"`）。
- **`rails db:migrate:status`** で適用済み/未適用のマイグレーションを確認。
- **本番 console は慎重に**：`update_all` / `delete_all` は確認や検証なしで走る。先に `count` で件数確認。
- Rails 5 では `rake` も残るが、**新規は `rails` コマンド**で統一するのが推奨。

## ハマりどころ / アンチパターン
- **本番 console で破壊操作**：`User.delete_all` などは即実行。`--sandbox` や `transaction do ... raise end` で守る。
- **`save` の戻り値を見ない**：console で `u.save` が `false` を返しても気づかず「保存したつもり」。`save!` か `errors` を確認。
- **`reload!` 忘れ**：コードを直したのに console が古い定義のまま。`reload!` するか console を入れ直す。
- **環境取り違え**：`-e production` を付けたまま開発操作をして本番データを触る。プロンプトの環境表示を確認。

## 関連
[active_record.md](./active_record.md) / [getting_started.md](./getting_started.md)
