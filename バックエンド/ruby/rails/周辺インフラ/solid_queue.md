# Solid Queue（Rails の関連事項 / 周辺インフラ）

## ひとことで言うと
**データベースをバックエンドにする Active Job 実行基盤**。Redis を立てずに、いつものSQL DB（PostgreSQL / MySQL / SQLite）だけでバックグラウンドジョブを回せる。**Rails 8 の既定**ジョブバックエンド。

## どの版で使えるか
- **Rails 8**: 既定（`rails new` で同梱・設定済み）。
- **Rails 7.1〜7.2**: gem を追加すれば利用可（`gem "solid_queue"` ＋ インストールタスク）。
- DHH/37signals 製の「**Solid トリオ**」の1つ：**Solid Queue（ジョブ）/ Solid Cache（キャッシュ）/ Solid Cable（Action Cable）**。3つ揃うと「Redis無しでRailsを本番運用」が現実的になる。

## 役割・なぜ必要か
- **インフラを増やしたくない**ニーズに応える。Sidekiq は速いが Redis の運用・監視・コストが付く。Solid Queue は「DBはどうせ要る」ので**追加ミドルウェアゼロ**でジョブを実現する。
- ジョブのenqueueをアプリのDBトランザクションと**同じDB**に乗せられる＝整合性を取りやすい。

## 基本の使い方（コード）
```ruby
# config/application.rb（または環境別）
config.active_job.queue_adapter = :solid_queue

# ジョブ自体は Active Job なので書き方は共通
class ReportJob < ApplicationJob
  def perform(user_id) ; end
end
ReportJob.perform_later(user.id)
```
```bash
bin/jobs            # ワーカー起動（Rails 8 同梱のランナー）
# Puma 内で動かす構成（小規模向け）: SOLID_QUEUE_IN_PUMA=true
```
- 仕組み: `FOR UPDATE SKIP LOCKED`（行ロックの取り合いを避けるSQL機能）でワーカーがジョブを奪い合う。専用テーブル群を持つ。

## 実務での勘所
- **小〜中規模・低〜中スループット**に好適。インフラ簡素化・コスト削減が効く。SQLiteでの本番運用（Rails 8 の流れ）とも相性が良い。
- 専用DBや別コネクションプールに分けると、アプリ本体のDB負荷と切り離せる。
- リトライ・並行度・キュー優先度・定期実行（recurring）を設定で持つ。

## ハマりどころ / 注意
- **DBへの負荷**：ジョブのポーリング/ロックがDBに乗る。超高スループットでは Sidekiq + Redis の方が有利。
- ワーカープロセス（`bin/jobs`）の起動・常駐をデプロイに組み込む必要（Puma内実行は手軽だが本番規模では分離推奨）。
- 既存アプリの移行では、ジョブテーブルのマイグレーションと adapter 切替を段階的に。

## Sidekiq との使い分け
| | Solid Queue | Sidekiq |
|---|---|---|
| 土台 | DB（追加不要） | Redis（別途必要） |
| 規模 | 小〜中 | 中〜大・高スループット |
| 運用 | 簡素 | Redis運用が要る |
| 既定 | Rails 8 | gem導入 |

→ [sidekiq.md](./sidekiq.md) / [redis.md](./redis.md)

## 関連
[sidekiq.md](./sidekiq.md) / [redis.md](./redis.md)
