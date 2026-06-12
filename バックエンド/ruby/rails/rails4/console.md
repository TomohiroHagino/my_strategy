# rails console / コマンド（Rails 4）

## ひとことで言うと
`rails console` はアプリのコードを読み込んだ状態でRubyを対話実行できる場、`rake` はマイグレーションやタスクを実行するコマンド群。Rails 4 では **`rake` が主流**（`bin/rails` も使える）。

## 役割・なぜ必要か
- モデルの挙動確認・データ調査・一時的なデータ修正を、アプリの全コードを読み込んだ状態で手早く試すために console がある。
- DB操作・カスタム処理をコマンド化して再実行可能にするために rake タスクがある。

## 基本の書き方（コマンド）

### console
```bash
rails console            # = rails c。開発環境で起動
rails console production # 本番環境（要注意）
rails console --sandbox  # 終了時に全変更をロールバック（調査向け・安全）
```
```ruby
# console 内
User.count
u = User.find(1)
u.update(name: "新名")      # その場で更新（本番では特に慎重に）
reload!                      # コード変更を再読込
Post.where("created_at > ?", 1.day.ago).pluck(:id)
app.get "/posts"            # 統合テスト風にリクエストを送れる
helper.number_to_currency(1000)  # ヘルパーも呼べる
```

### rake（主流） / コマンド
```bash
rake db:create db:migrate db:seed      # DB作成→マイグレ→初期データ
rake db:rollback STEP=2                 # 2つ戻す
rake db:migrate:status                  # 適用状況
rake routes                             # ルート一覧（rails routes は5系から）
rake -T                                 # タスク一覧
rake secret                             # secret_key_base 用のランダム値生成
rails generate model Post title:string  # ジェネレータ
rails server                            # サーバ起動（rails s）
rails dbconsole                         # DBのCLIに直接入る
```

### カスタム rake タスク
```ruby
# lib/tasks/cleanup.rake
namespace :cleanup do
  desc "古い下書きを削除"
  task old_drafts: :environment do      # :environment でアプリを読み込む
    Post.where(published: false).where("created_at < ?", 1.month.ago).find_each(&:destroy)
  end
end
# 実行: rake cleanup:old_drafts
```

## 実務での使い方・定番パターン
- **調査は `--sandbox`**：終了時ロールバックで本番データを汚さず確認。
- **一括データ修正は `find_each`** でバッチ処理（全件メモリ展開を避ける）。
- **本番での変更は対象と件数を先に確認**：`scope.count` → 想定通りなら実行。`update_all` は速いがコールバック/バリデーションを通さない点に注意。→ [active_record.md](./active_record.md)
- **再実行する処理は rake タスク化**：`task: :environment` を付けてモデルを使えるように。
- `app` / `helper` オブジェクトでリクエストやヘルパーを console から試せる。

## ハマりどころ / アンチパターン
- **本番 console での無防備な変更**：`User.update_all(...)` のような全件操作で事故。`--sandbox`・件数確認・`where` 限定を徹底。
- **`rails routes` を叩く**：Rails 4 は `rake routes`（`rails routes` は5系）。
- **`update_all` / `delete_all` でコールバック非実行**：関連削除や整合性処理が走らない。→ [active_record.md](./active_record.md)
- **`reload!` 忘れ**：console 起動中にコードを直しても反映されない。`reload!` するか開き直す。
- **`:environment` 付け忘れ**：rake タスクからモデルを呼ぶと `uninitialized constant`。`task x: :environment do` に。

## 関連
[active_record.md](./active_record.md) / [getting_started.md](./getting_started.md) / [pitfalls.md](./pitfalls.md)
