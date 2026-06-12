# Redis（Rails 7 の関連事項）

## ひとことで言うと
**インメモリ（メモリ上）のキー・バリュー型データストア**。超高速だが揮発性（基本はメモリ、永続化も可能）。Rails本体ではないが、実務のRailsアプリでほぼ必ず脇に立つミドルウェア。

## Rails で何に使うか（役割）
1. **ジョブキューの土台**：Sidekiq / Resque がジョブを積む先。→ [sidekiq.md](./sidekiq.md)
2. **キャッシュストア**：`config.cache_store = :redis_cache_store`。重い計算やDB結果を一時保存。
3. **Action Cable の pub/sub**：複数プロセス/サーバ間でWebSocketメッセージを配る土台。→ [Hotwire（各版）](../rails7/hotwire.md)（Turbo Streamのリアルタイム配信）
4. **セッションストア**（cookie に載せたくない大きめの状態を持つ場合）。
5. **レート制限・カウンタ・ランキング**（`INCR`、Sorted Set など）、分散ロック。

## 基本の使い方（コード）
```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, { url: ENV["REDIS_URL"] }

# 低レベルキャッシュ
Rails.cache.fetch("user_#{id}_stats", expires_in: 5.minutes) do
  heavy_calculation
end

# 直接叩く（redis gem）
$redis = Redis.new(url: ENV["REDIS_URL"])
$redis.incr("page_views")
```

## 実務での勘所
- **TTL（有効期限）設計**が肝。`expires_in` を付けないと無限にメモリを食う。
- **メモリは有限**：`maxmemory` と追い出しポリシー（`allkeys-lru` 等）を理解する。
- **永続化**（RDB スナップショット / AOF）の要否を用途で判断（キャッシュなら飛んでも可、ジョブなら飛ぶと困る）。
- マネージドサービス（AWS ElastiCache / GCP Memorystore / Redis Cloud）で運用負荷を下げるのが一般的。

## ハマりどころ
- **「Redisが落ちる＝アプリも落ちる」依存**：キャッシュ用途なら fail-safe（落ちても素通りで動く）に。`error_handler` を設定。
- **キャッシュキーの設計ミス**で古いデータが残る/衝突する。バージョンやモデルの `cache_key_with_version` を活用。
- 巨大な値や `KEYS *` の本番実行でブロッキング。
- Sidekiq と キャッシュで**同じRedisを共用すると競合**しがち。用途ごとにDB番号やインスタンスを分ける。

## Solid Queue との関係
Rails 8 系では **Solid Queue / Solid Cache / Solid Cable** によって「**Redisを使わずDBだけ**」でジョブ・キャッシュ・Cableを賄う選択肢が標準化された。小〜中規模では「Redisを増やさない」構成が現実的に。→ [solid_queue.md](./solid_queue.md)

## 関連
[sidekiq.md](./sidekiq.md) / [solid_queue.md](./solid_queue.md) / [Hotwire（各版）](../rails7/hotwire.md)
