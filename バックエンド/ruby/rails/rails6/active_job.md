# Active Job（Rails 6）

## ひとことで言うと
時間のかかる処理をバックグラウンドで非同期実行するための共通インターフェース。Sidekiq等のバックエンドをアダプタ経由で差し替えられ、メール送信の `deliver_later` もこの上で動く。

## 役割・なぜ必要か
リクエスト内で重い処理（メール送信・画像変換・外部API呼び出し）を同期実行するとレスポンスが遅くなる。ジョブとしてキューに積み、別プロセスで処理することでリクエストを即返せる。アダプタ（sidekiq/resque/async等）を切り替えてもジョブのコードは同じ。

## 基本の書き方（コード）
ジョブ生成: `rails g job send_notification`
```ruby
class SendNotificationJob < ApplicationJob
  queue_as :default

  retry_on Net::OpenTimeout, wait: 5.seconds, attempts: 3
  discard_on ActiveJob::DeserializationError  # 対象消滅などは破棄

  def perform(user, message)  # 引数はGlobalIDで自動シリアライズ
    user.notify(message)
  end
end
```
実行:
```ruby
SendNotificationJob.perform_later(user, "hello")  # キューに積む（非同期）
SendNotificationJob.perform_now(user, "hello")     # その場で同期実行
SendNotificationJob.set(wait: 1.hour).perform_later(user, "後で")
```
本番アダプタ設定（`config/environments/production.rb`）:
```ruby
config.active_job.queue_adapter = :sidekiq
# 開発の既定は :async（プロセス内スレッド、再起動で消える）
```

## 実務での使い方・定番パターン
- メール送信: `UserMailer.welcome(user).deliver_later` は内部で `ActionMailer::MailDeliveryJob` を `perform_later` する（[action_mailer.md](./action_mailer.md)）。
- 重い処理の非同期化: CSVエクスポート・画像リサイズ・外部API同期などを `perform_later` に逃がす。
- 本番は Sidekiq + Redis が定番。詳細は [`../周辺インフラ/sidekiq.md`](../周辺インフラ/sidekiq.md) と [`../周辺インフラ/redis.md`](../周辺インフラ/redis.md)。
- 冪等性: ジョブは再実行されうる前提で書く。「未送信なら送る」のように状態を見て二重実行に耐える設計にする。
```ruby
def perform(order)
  return if order.notified?      # すでに処理済みなら何もしない
  order.deliver_notification!
  order.update!(notified: true)
end
```

## ハマりどころ / アンチパターン
- 引数にAR オブジェクトを渡すとGlobalIDでID参照に変換され、実行時に再ロードされる（DB1件分のクエリ）。巨大なオブジェクトや配列を直接渡さず、IDや必要な値だけ渡す。
- 本番でアダプタ未設定だと `:async`（あるいは古い設定で `:inline`）になり、別プロセスで動かない／再起動で失われる。`queue_adapter = :sidekiq` を明示する。
- `retry_on` の `attempts` を無制限にすると失敗ジョブが延々リトライしてキューを圧迫する。回数上限と `discard_on` を設定する。
- `perform_later` の引数はシリアライズ可能なもの（プリミティブ・GlobalID対象）に限る。シリアライズできない値で例外。
- ジョブ内で例外を握りつぶすとリトライ機構が働かない。再送すべきは raise する。

## 関連
[action_mailer.md](./action_mailer.md) / [`../周辺インフラ/sidekiq.md`](../周辺インフラ/sidekiq.md) / [`../周辺インフラ/redis.md`](../周辺インフラ/redis.md) / [active_record.md](./active_record.md) / [config_credentials.md](./config_credentials.md) / [pitfalls.md](./pitfalls.md)
