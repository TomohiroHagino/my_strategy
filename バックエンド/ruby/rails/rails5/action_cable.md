# Action Cable（Rails 5 特有）

## ひとことで言うと
**WebSocket を使ったリアルタイム双方向通信を Rails に統合する仕組み**。チャット・通知・ライブ更新などを、サーバからクライアントへ即時にデータを push できる。Rails 5 で標準同梱された。

## 役割・なぜ必要か
- 通常のHTTPは「クライアントが要求→サーバが応答」の一方向。サーバ側のイベント（新着メッセージ等）をリアルタイムに画面へ届けるには WebSocket が要る。
- Action Cable は WebSocket 接続の管理・チャンネル・購読・ブロードキャストを Rails の枠組み（コントローラに似た Channel）で書けるようにする。
- 複数プロセス/サーバ間でブロードキャストを共有するため、**Redis を pub/sub バックエンドとして使う**のが本番の定番。→ [../周辺インフラ/redis.md](../周辺インフラ/redis.md)

## 構成要素
- **Connection**（`app/channels/application_cable/connection.rb`）：接続単位。ここで接続元ユーザを認証する。
- **Channel**（`app/channels/*_channel.rb`）：購読の単位（チャットルームなど）。サーバ側ロジック。
- **Subscription**（`app/javascript/channels/*.js` 等）：クライアント側で購読し、受信時の処理を書く。
- **Broadcasting**：サーバから特定ストリームへデータを送る。

## 基本の書き方（コード）
```ruby
# config/routes.rb
mount ActionCable.server => "/cable"
```
```ruby
# app/channels/application_cable/connection.rb（接続の認証）
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user
    def connect
      self.current_user = User.find_by(id: cookies.signed[:user_id]) || reject_unauthorized_connection
    end
  end
end
```
```ruby
# app/channels/room_channel.rb
class RoomChannel < ApplicationCable::Channel
  def subscribed
    stream_from "room_#{params[:room_id]}"     # このストリームを購読
  end
  def unsubscribed; end
end
```
```ruby
# サーバ側からブロードキャスト（コントローラやジョブから）
ActionCable.server.broadcast("room_#{room.id}", message: rendered_html)
```
```js
// クライアント側（Rails 5.0/5.1 は coffee/JS、5.1+ は Webpacker でも書ける）
App.room = App.cable.subscriptions.create({ channel: "RoomChannel", room_id: 1 }, {
  received(data) { /* 受信時に画面更新 */ }
})
```
```yaml
# config/cable.yml（本番は redis）
production:
  adapter: redis
  url: redis://localhost:6379/1
```

## 実務での使い方・定番パターン
- **接続認証を Connection で必ず行う**：`reject_unauthorized_connection` で未ログイン接続を弾く。
- **ブロードキャストはジョブから**：重い描画やブロードキャストは Active Job に逃がす。→ [active_job.md](./active_job.md)
- **本番は Redis アダプタ**：複数 Puma ワーカー/複数サーバ間で購読を共有するため必須。`async` アダプタは単一プロセスのみ。→ [../周辺インフラ/redis.md](../周辺インフラ/redis.md)
- **`allowed_request_origins`** を設定して、想定外オリジンからの WebSocket 接続を拒否する。

## ハマりどころ / アンチパターン
- **本番で `async` アダプタのまま**：プロセスをまたいでブロードキャストが届かない（片方のワーカーにしか飛ばない）。本番は `redis`。
- **接続認証の漏れ**：Connection で認証しないと誰でも購読できる。`identified_by` ＋ `reject_unauthorized_connection`。
- **オリジン制限の未設定**：`allowed_request_origins` を絞らないと別サイトから接続される。
- **WebSocket をスケールさせる前提を忘れる**：常時接続はプロセス/ファイルディスクリプタを消費する。接続数が多いなら専用サーバ・前段の検討が要る。
- **Webpacker 有無でフロントの置き場が変わる**：5.0 は `app/assets/javascripts/channels`、Webpacker 導入時は `app/javascript/channels`。

## 関連
[active_job.md](./active_job.md) / [../周辺インフラ/redis.md](../周辺インフラ/redis.md) / [routing.md](./routing.md)
