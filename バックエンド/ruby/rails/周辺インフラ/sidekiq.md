# Sidekiq（Rails の関連事項 / 周辺インフラ）

## ひとことで言うと
**Redis を土台にした、Rails 定番のバックグラウンドジョブ実行基盤**。「重い/遅い処理」をリクエストの外（別プロセス）で非同期に走らせる。

## 役割・なぜ必要か
- メール送信・画像処理・外部API呼び出し・集計など、レスポンスを待たせたくない処理を**後回し**にしてユーザー体験を守るためにある。
- マルチスレッドで動くため、プロセスあたりの並列度が高く効率的（Resque/DelayedJobより高スループット）。

## 基本の使い方（コード）
```ruby
# Gemfile: gem "sidekiq"

# Active Job 経由（バックエンドにSidekiqを指定）
class ThumbnailJob < ApplicationJob
  queue_as :default
  def perform(image_id)
    Image.find(image_id).generate_thumbnail!
  end
end
ThumbnailJob.perform_later(image.id)   # キューに積む

# Sidekiq::Worker を直接使う書き方もある
class HardWorker
  include Sidekiq::Job
  def perform(id) ; end
end
```
```bash
bundle exec sidekiq          # ワーカープロセス起動（Redisが必要）
```
- 管理画面（Web UI）でキュー・失敗・リトライを可視化できる。

## 実務での勘所
- **ジョブ引数はAR本体でなくID**を渡す（シリアライズ問題＆データ鮮度のため）。
- **冪等性**：同じジョブが2回実行されても壊れない設計に（リトライ前提）。
- **リトライ/デッドセット**：失敗時の再試行回数とバックオフを設計。最終失敗の監視を。
- **キュー分割**（`critical` / `default` / `low`）で優先度制御。
- 定期実行は sidekiq-cron / sidekiq-scheduler。

## ハマりどころ
- **Redis 依存**：Redisが落ちるとジョブが積めない/消える。永続化と監視が要る。→ [redis.md](./redis.md)
- **デプロイ時の取りこぼし**：稼働中ジョブのグレースフル停止（`-t` タイムアウト）。
- メモリ肥大（長時間プロセス）でのリーク。
- トランザクション内で `perform_later` すると、コミット前にジョブが走って「まだ無いレコード」を引く → `after_commit` で積むか Active Job の遅延を使う。

## 代替・比較
- **Solid Queue**（Redis不要・DBバックエンド、Rails 8 既定）→ [solid_queue.md](./solid_queue.md)
- GoodJob（Postgres専用・DBバックエンド）、Resque/DelayedJob（旧来）。
- 選び方: 高スループット重視→Sidekiq、インフラを増やしたくない→Solid Queue/GoodJob。

## 関連
[redis.md](./redis.md) / [solid_queue.md](./solid_queue.md)
