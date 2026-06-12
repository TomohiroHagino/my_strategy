# 複数データベース（Rails 6）

## ひとことで言うと
Rails 6で標準化された複数DB接続。`database.yml` に primary/replica を書き、`connects_to`/`connected_to` で読み書きを振り分けられる。HTTPメソッドに応じた自動role切替や、6.1の水平シャーディングも使える。

## 役割・なぜ必要か
読み取り負荷をレプリカDBに逃がしてプライマリの負荷を下げたり、用途別にDBを分割（例: アプリ用とログ用）したりできる。6以前はgem頼みだった機能がフレームワーク標準になった。

## 基本の書き方（コード）
`config/database.yml`（primaryと読み取り用replica）:
```yaml
production:
  primary:
    database: app_production
    adapter: mysql2
  primary_replica:
    database: app_production       # 同一データのレプリカ
    adapter: mysql2
    replica: true                  # 書き込み不可・マイグレーション対象外
    database_tasks: false          # db:migrate等の対象から除外
```
抽象クラスで接続を宣言:
```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :primary, reading: :primary_replica }
end
```
明示的な切り替え:
```ruby
ActiveRecord::Base.connected_to(role: :reading) do
  User.count          # replica から読む
end

ActiveRecord::Base.connected_to(role: :writing) do
  User.create!(name: "a")  # primary に書く
end
```
自動role切替（GET/HEADは reading、それ以外は writing）:
```ruby
# config/environments/production.rb
config.active_record.database_selector = { delay: 2.seconds }
config.active_record.database_resolver = ActiveRecord::Middleware::DatabaseSelector::Resolver
config.active_record.database_resolver_context = ActiveRecord::Middleware::DatabaseSelector::Resolver::Session
```

## 実務での使い方・定番パターン
- 読み取りをレプリカへ: 集計や一覧など書き込みを伴わない処理を `connected_to(role: :reading)` で囲む。
- 用途別DB分割: ログや分析用テーブルを別DBにし、専用の抽象クラスを継承させる。
```ruby
class LogRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to database: { writing: :logs }
end
class AccessLog < LogRecord; end  # logs DB に接続
```
- 水平シャーディング（6.1）: 同じスキーマを複数DBに分割。
```ruby
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true
  connects_to shards: {
    default:   { writing: :primary,   reading: :primary_replica },
    shard_one: { writing: :primary_2, reading: :primary_2_replica }
  }
end
ActiveRecord::Base.connected_to(shard: :shard_one) { User.find(1) }
```

## ハマりどころ / アンチパターン
- 書き込み直後にレプリカから読むと**レプリカ遅延**で古いデータを引く。`database_selector` の `delay` 内は writing に固定される。直後の整合性が必要なら明示的に `connected_to(role: :writing)`。
- マイグレーションは `replica: true` / `database_tasks: false` のDBを対象外にする。これを付け忘れるとレプリカにmigrateしようとして失敗する。
- 抽象クラス（`abstract_class = true` ＋ `connects_to`）の継承を忘れると、そのモデルだけ別DBに繋がらず想定外の接続になる。
- `connected_to` のブロックを抜けると元のroleに戻る。ブロック外で読み書きを期待しない。
- シャーディングで `connected_to(shard:)` の指定漏れは default シャードに行く。対象シャードを必ず明示する。

## 関連
[active_record.md](./active_record.md) / [model.md](./model.md) / [config_credentials.md](./config_credentials.md) / [`../周辺インフラ/redis.md`](../周辺インフラ/redis.md) / [console.md](./console.md) / [pitfalls.md](./pitfalls.md)
