# Active Job（バックグラウンドジョブ）（Rails 4）

## ひとことで言うと
時間のかかる処理（メール送信・画像変換・外部API呼び出し）を**リクエストの外で非同期に実行する仕組み**。**Active Job は Rails 4.2 で導入**され、Sidekiq/Resque/DelayedJob などのジョブ基盤を**共通のインターフェースで抽象化**する。

## 役割・なぜ必要か
- リクエスト内で重い処理をやるとユーザを待たせるので、キューに積んで別プロセス（ワーカ）で後から実行するためにある。
- 4.2 以前は Sidekiq なら `Sidekiq::Worker`、Resque なら別の書き方…とバックエンドごとに書き方が違った。Active Job が**バックエンドを差し替え可能な共通APIを提供**することで、アプリ側のジョブコードを統一できる。

## 重要な前提（バージョン）
- **Active Job が使えるのは Rails 4.2 以降**。
- **Rails 4.0 / 4.1 には Active Job が無い**。これらでは Sidekiq 等を**直接**使う（後述）。`app/jobs/` ディレクトリも 4.2 から。

## 基本の書き方（コード／4.2〜）
```ruby
# app/jobs/thumbnail_job.rb
class ThumbnailJob < ActiveJob::Base       # 4.2は ActiveJob::Base を継承
  queue_as :default

  def perform(image_id)
    image = Image.find(image_id)
    image.generate_thumbnail!
  end
end
```
```ruby
# 実行（キューに積む）
ThumbnailJob.perform_later(image.id)             # 非同期
ThumbnailJob.set(wait: 5.minutes).perform_later(image.id)  # 遅延実行
ThumbnailJob.perform_now(image.id)               # 同期（その場で実行）
```

### バックエンド設定（4.2〜）
```ruby
# config/application.rb
config.active_job.queue_adapter = :sidekiq   # :inline / :async / :resque など
```
- 既定は `:inline`（その場で同期実行＝開発向け）。本番は Sidekiq などへ。→ [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md)
- メールの `deliver_later` は内部で Active Job を使う。→ [action_mailer.md](./action_mailer.md)

## 4.0 / 4.1 で非同期処理をするには（Active Job が無い場合）
```ruby
# Sidekiq を直接使う
class ThumbnailWorker
  include Sidekiq::Worker
  def perform(image_id)
    Image.find(image_id).generate_thumbnail!
  end
end

ThumbnailWorker.perform_async(image.id)   # キューに積む
```
- 4.2 へ上げると `ActiveJob::Base` に寄せられる。新規 4.2 プロジェクトは最初から Active Job 経由を推奨。

## 実務での使い方・定番パターン
- **引数はIDなどプリミティブを渡す**：ActiveRecord オブジェクトを直接渡すとシリアライズ/古いデータ参照の問題。ジョブ内で `find` し直す。（4.2 の GlobalID でオブジェクトも渡せるが、ID渡しが安全。）
- **冪等に書く**：同じジョブが2回走っても壊れないように（リトライ前提）。
- **失敗時のリトライ/デッドレター**は Sidekiq 側の機能を使う。→ [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md)
- **キュー分割**：重要度/処理時間で `queue_as` を分ける。
- Redis が Sidekiq の土台。→ [../周辺インフラ/redis.md](../周辺インフラ/redis.md)

## ハマりどころ / アンチパターン
- **4.0/4.1 で Active Job を使おうとする**：存在しない。Sidekiq 等を直接。バージョンを必ず確認。
- **オブジェクトをそのまま引数に**：データが古くなる/巨大化。IDを渡してジョブ内で再取得。
- **`perform_now` のつもりで本番が `:inline`**：開発の `:inline` 設定のまま本番に出すと非同期にならずリクエストが遅い。本番は Sidekiq 等に。
- **ジョブ内の例外を握り潰す**：失敗が見えない。リトライ設定とログ/監視を入れる。
- **長時間ジョブを1本に**：途中失敗で全やり直し。分割して冪等に。

## 関連
[action_mailer.md](./action_mailer.md) / [service_form.md](./service_form.md) / [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md) / [../周辺インフラ/redis.md](../周辺インフラ/redis.md)
