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

## Redis に直接書き込む（`Rails.cache` を介さず）
`Rails.cache.fetch` はキャッシュ抽象（キーにプレフィックスや名前空間が付く）。**カウンタ・レート制限・ランキング・分散ロック**のように「素のRedisコマンドを直接叩きたい」ときは、Redisクライアントを直接使う。

### ① 接続を1か所に用意する（initializer）
毎回 `Redis.new` するとコネクションが増えて枯渇する。**接続プール**で共有するのが定石。
```ruby
# Gemfile
gem "redis"
gem "connection_pool"

# config/initializers/redis.rb
REDIS = ConnectionPool.new(size: 5, timeout: 3) do
  Redis.new(url: ENV.fetch("REDIS_URL"))
end
```

### ② 直接コマンドを叩く（TTL必須）
```ruby
REDIS.with do |r|
  # カウンタ（アトミックに +1）
  r.incr("page_views:#{Date.current}")

  # 文字列＋有効期限（秒）。直接書くキーには必ずTTLを付ける
  r.set("flag:maintenance", "1", ex: 600)   # 600秒で自動消滅
  r.setex("session:#{sid}", 3600, payload)  # 上と同義

  # 構造を使う
  r.hset("user:#{id}", "name", "Alice", "age", 30)   # Hash
  r.lpush("queue:mail", job_json)                    # List（キュー）
  r.zadd("ranking", score, user_id)                  # Sorted Set（ランキング）
  r.zrevrange("ranking", 0, 9, with_scores: true)    # 上位10件
end
```

### ③ よくある実務パターン
```ruby
# レート制限（1分に5回まで）: INCR + 初回だけ EXPIRE
def rate_limited?(key, limit: 5, period: 60)
  REDIS.with do |r|
    count = r.incr(key)
    r.expire(key, period) if count == 1   # 最初の1回でTTLを張る
    count > limit
  end
end

# Rails.cache 経由でも生Redisに触れる（redis_cache_store 利用時）
Rails.cache.redis.with { |r| r.dbsize }
```

### Rails.cache と直接書きの使い分け
| 用途 | 使うもの |
|---|---|
| DB結果・計算結果のキャッシュ | `Rails.cache.fetch`（抽象・fail-safe が効く） |
| カウンタ / レート制限 / ランキング / ロック | **生Redis（直接）**（`INCR`/`ZADD` 等が必要） |
| ジョブ投入 | Sidekiq 等のクライアント経由（自分で `LPUSH` しない） |

> **注意**：直接書いたキーは `Rails.cache` の名前空間の**外**に出る。`Rails.cache.clear` では消えないので、TTLか明示 `DEL` で寿命を管理する。Sidekiq と**同じRedis/DB番号を共用しない**（誤 `FLUSHDB` やキー衝突の元）。

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
Redisエンジンそのもの（データ型・redis-cli・永続化・他言語クライアント）→ [../../../../データベース/redis.md](../../../../データベース/redis.md)
