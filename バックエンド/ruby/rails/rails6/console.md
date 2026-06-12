# rails console / コマンド（Rails 6）

## ひとことで言うと
`rails console` はアプリを読み込んだ対話環境で、ActiveRecord を直接叩いてデータ確認・修正ができる。他にも migrate / routes / runner などの実務コマンドがある。

## 役割・なぜ必要か
「このユーザーのレコードを確認したい」「1回だけデータを直したい」ときに、わざわざ画面やスクリプトを作らずコンソールで `User.find(1)` を実行できる。バッチ的な一発実行は `rails runner`、ルート確認は `rails routes` と使い分ける。

## 基本の書き方（コード）
```bash
rails console                 # 開発環境のコンソール（rails c）
rails console -e production    # 本番環境を読み込んで起動
rails console --sandbox        # 終了時に全変更をロールバックする安全モード

rails runner "User.where(active: false).update_all(active: true)"  # 一発実行
rails dbconsole                # DB の対話シェル（psql/mysql 等）
rails routes                   # ルーティング一覧（grep と併用）
rails routes | grep post
```
DB・生成系。
```bash
rails db:migrate               # マイグレーション適用
rails db:rollback STEP=1       # 1つ戻す
rails db:seed                  # seeds.rb 実行
rails -T                       # rake/rails タスク一覧
rails generate model Post title:string   # 生成（rails g）
rails destroy model Post                  # 生成物の取り消し
```
コンソール内のよく使う操作。
```ruby
reload!                       # コード変更をコンソールに反映
Post.count
Post.where(published: true).order(created_at: :desc).limit(5)
post = Post.find(1); post.update(title: "新タイトル")
```

## 実務での使い方・定番パターン
- 本番でデータを確認するときは `rails console -e production --sandbox` で起動し、まず参照・検証してから本番反映する。
- 定期処理や一括更新の単発実行は `rails runner`（cron からも呼べる）。
- ルート名・パスが分からないときは `rails routes | grep xxx`。

## ハマりどころ / アンチパターン
- 本番コンソールでの誤操作（`destroy_all` 等）は即データ損失。まず `--sandbox` で挙動を確認する。
- コンソール起動中にコードを修正しても自動反映されない。`reload!` を忘れると古い挙動のまま。
- `--sandbox` を抜けると、そのセッションで保存した変更も全てロールバックされる。本当に反映したい変更を sandbox 内でやらない。

## 関連
[getting_started.md](./getting_started.md) / [active_record.md](./active_record.md) / [model.md](./model.md) / [config_credentials.md](./config_credentials.md) / [multiple_databases.md](./multiple_databases.md) / [pitfalls.md](./pitfalls.md)
