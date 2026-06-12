# Active Job（バックグラウンドジョブ）（Rails 5）

## ひとことで言うと
**バックグラウンド処理の共通インターフェース**。「重い/遅い処理」をリクエストの外（別プロセス）で非同期に実行する仕組みで、**実行基盤（Sidekiq / Resque など）を差し替えても書き方は同じ**にできる抽象レイヤ。Rails 5 からは `ApplicationJob` を継承する。

## 役割・なぜ必要か
- メール送信・画像処理・外部API呼び出し・集計などをレスポンスから切り離し、**ユーザーを待たせない**ためにある。
- Active Job は「キューに積む API」を統一し、裏側のアダプタ（Sidekiq / Resque / DelayedJob …）を**設定1行で交換可能**にする。アプリのジョブコードはアダプタに依存しない。

## 基本の書き方（コード）
```ruby
# app/jobs/application_job.rb（Rails 5 で導入された共通基底）
class ApplicationJob < ActiveJob::Base
end

# rails g job thumbnail で生成
# app/jobs/thumbnail_job.rb
class ThumbnailJob < ApplicationJob
  queue_as :default                 # 積むキュー名（critical/default/low 等で優先度分け）
  retry_on Net::OpenTimeout, wait: 5.seconds, attempts: 5  # 一時的失敗はリトライ（5.1+）
  discard_on ActiveJob::DeserializationError               # 復元不能なら捨てる（5.1+）

  def perform(image_id)             # ★ 引数はAR本体でなく ID を渡す
    image = Image.find(image_id)
    image.generate_thumbnail!
  end
end

# 呼び出し
ThumbnailJob.perform_later(image.id)   # キューに積む（非同期・通常はこちら）
ThumbnailJob.perform_now(image.id)     # その場で同期実行（テスト/即時処理用）
ThumbnailJob.set(wait: 10.minutes).perform_later(image.id)  # 遅延実行
```
```ruby
# config/environments/production.rb（アダプタの指定。差し替えはここ1行）
config.active_job.queue_adapter = :sidekiq      # または :resque など
# 開発/テスト規定: :async（プロセス内スレッド・Rails 5 で既定化）/ test では :test
```

## 実務での使い方・定番パターン
- **引数はIDを渡す**：AR本体を渡すとシリアライズ問題＋実行時にデータが古い。`perform` 内で `find` し直す（GlobalID で本体も渡せるが ID 推奨）。
- **冪等性**：同じジョブが2回走っても壊れない設計に（リトライ・重複enqueue前提）。処理済みフラグやユニーク制約で二重実行を吸収。
- **リトライ設計**：`retry_on`（一時障害）と `discard_on`（恒久失敗）を使い分け（どちらも 5.1+）。最終失敗の監視・通知を必ず用意。
- **キュー分割**：`queue_as :critical` などで優先度制御。緊急ジョブが低優先に埋もれないように。
- **バックエンドは Sidekiq が定番**（Redis前提）→ [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md) / [../周辺インフラ/redis.md](../周辺インフラ/redis.md)

## ハマりどころ / アンチパターン
- **トランザクション内で `perform_later`（最頻）**：コミット前にワーカーが走り、「まだDBに無いレコード」を引いて失敗する。→ **`after_commit` で積む**か、トランザクション外へ出す。
  ```ruby
  after_commit :enqueue_thumbnail, on: :create
  def enqueue_thumbnail
    ThumbnailJob.perform_later(id)
  end
  ```
- **`perform_now` を本番のリクエスト中で多用**：結局リクエストをブロックして非同期の意味がなくなる。
- **ジョブにAR本体を渡す**：シリアライズ肥大・鮮度劣化・デプロイ跨ぎでの復元失敗。
- **`retry_on`/`discard_on` は 5.1 から**：5.0 では使えない。5.0 はアダプタ側（Sidekiq）のリトライ機構に頼る。
- **アダプタ未起動**：`:async` 以外はワーカープロセスの常駐が必要（Sidekiq の `bundle exec sidekiq`）。

## 関連
[action_mailer.md](./action_mailer.md) / [../周辺インフラ/sidekiq.md](../周辺インフラ/sidekiq.md) / [../周辺インフラ/redis.md](../周辺インフラ/redis.md)
