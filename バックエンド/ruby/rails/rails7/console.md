# rails console / コマンド（Rails 7）

## ひとことで言うと
`rails console`（略 `rails c`）は、**アプリ本体を読み込んだ状態のIRB対話環境**。モデル・サービス・ヘルパーを実際のコードのまま呼び出して、データ確認・動作検証・調査ができる。

## 役割・なぜ必要か
- 「このクエリで何件返る？」「この値で `save!` は通る？」といった確認を、画面やコードを書かずに**その場で試す**ためにある。
- バグ調査・データ調査・1回限りの修正・新しいAPIの素振りに使う、Rails開発の日常ツール。

## 基本の書き方（コード）
```bash
rails console          # 開発環境のコンソール起動（= rails c）
rails console -e production    # 本番環境を読み込んで起動
rails console --sandbox       # 終了時に全変更を自動ロールバック（安全）
```
```ruby
# コンソール内
User.count                       # 件数
u = User.find_by(email: "a@b.c") # 取得
u.posts.recent.limit(3)          # 関連・スコープもそのまま使える
reload!                          # コード変更をコンソールに再読込（再起動不要）
app.root_path                    # app 経由でルーティングヘルパーを叩く
app.get "/users"                 # 擬似リクエストでステータス確認
helper.number_to_currency(1000)  # helper 経由でビューヘルパーを呼ぶ
```

## 実務での使い方・定番パターン
- **`--sandbox`**：本番や共有DBで「読むだけ・試すだけ」のときは必ず付ける。終了時に**自動でロールバック**されるので破壊的変更が残らない。
- **`reload!`**：モデルを編集したらコンソールを再起動せず `reload!` で反映（既存変数は持ち越せないので取り直す）。
- **`app` / `helper`**：`app.xxx_path` でルーティング確認、`helper.xxx` でビューヘルパー検証。
- **主要コマンド**（`rails` 直下）:
  - `rails s`（server）/ `rails c`（console）
  - `rails g model Post title:string`（generate）/ `rails d`（destroy）
  - `rails db:migrate` / `db:rollback` / `db:seed` / `db:reset`
  - `rails routes`（ルート一覧、`-g posts` で絞り込み）
  - `rails runner "User.count"`（コンソールを開かず1行実行）
  - `bundle exec rspec`（テスト）
- **DB確認だけ**なら `rails dbconsole`（生のDBクライアントに入る）。

## ハマりどころ / アンチパターン
- **本番コンソールでの直接データ変更**：`update!` / `destroy!` / `delete_all` は取り返しがつかない。原則 `--sandbox`、変更が必要なら対象を `find` で確定→件数確認してから実行。
- **`--sandbox` の付け忘れ**：本番に入る瞬間にプロンプトの環境表示（赤字）を必ず確認する癖をつける。
- **`reload!` で旧オブジェクトが残る**：再読込後は変数を取り直さないと古いクラス定義のまま動く。
- **コールバックを通さない一括更新**：`update_all` / `delete_all` はバリデーション・コールバック非実行。意図しないデータ状態に注意。→ [active_record.md](./active_record.md)
- **`save` と `save!` の取り違え**：コンソールでは `save!` を使うと失敗理由（例外）がその場で見えて調査が速い。

## 関連
[active_record.md](./active_record.md) / [model.md](./model.md) / [routing.md](./routing.md)
