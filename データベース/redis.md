# Redis

## ひとことで言うと
**インメモリ（メモリ上）のキー・バリュー型データストア**。値をディスクではなく主にメモリに置くので**桁違いに速い**（読み書きがマイクロ秒〜）。キャッシュ・キュー・カウンタ・ランキング・pub/sub・分散ロックなど「速さと手軽さが要る脇役」の定番。

> RDB（MySQL/PostgreSQL）の代わりではなく**併用**するもの。永続的な正の在庫はRDB、速さが要る一時データはRedis、と役割を分ける。

## 役割・立ち位置
- **KVS（キー→値）** が基本。`SET key value` で書いて `GET key` で読む、という単純さが速さの源。
- 文字列だけでなく **Hash / List / Set / Sorted Set** などの構造を値に持てる（後述）。
- 主な用途：キャッシュ、ジョブキューの土台（Sidekiq等）、レート制限・カウンタ、ランキング（Sorted Set）、セッション、pub/sub、分散ロック。
- **揮発性が前提**（メモリ）。永続化もできるが「飛んでも作り直せるデータ」を置くのが基本思想。

---

## 直接書き込む方法（redis-cli / クライアントで直接操作）
RailsやLaravelのキャッシュ抽象（`Rails.cache` 等）を**通さず**、Redisへ直接キーを書く方法。デバッグ・初期投入・カウンタ・運用確認で多用する。

### ① redis-cli でつなぐ
```bash
# ローカル / コンテナ内
redis-cli                          # デフォルト localhost:6379 に接続
redis-cli -h 127.0.0.1 -p 6379     # ホスト/ポート指定
redis-cli -u redis://:password@host:6379/0   # URL指定（DB番号 0）

# つながったか確認
127.0.0.1:6379> PING
PONG
```

### ② 文字列を直接書く（一番基本）
```bash
SET user:101:name "Alice"          # 書き込み
GET user:101:name                  # 読み出し → "Alice"

SET page_views 0
INCR page_views                    # アトミックに +1（カウンタの定番）
INCRBY page_views 10               # +10

# 有効期限（TTL）つきで書く ← キャッシュは必ずこれ
SET session:abc "data" EX 3600     # 3600秒で自動消滅
SETEX session:abc 3600 "data"      # 上と同義
TTL session:abc                    # 残り秒数を確認（-1=無期限, -2=無い）
```

> **重要**：TTLを付けずに書くと**永遠にメモリに残る**。直接書くときほど `EX` を忘れない。

### ③ 構造ごとに直接書く
```bash
# Hash（1キーの中にフィールドを複数）= オブジェクトの保存に向く
HSET user:101 name "Alice" age 30
HGET user:101 name                 # "Alice"
HGETALL user:101                   # 全フィールド

# List（両端キュー）= ジョブキュー・最新N件
LPUSH queue:mail "job1"            # 左に積む
RPOP queue:mail                    # 右から取り出す

# Set（重複なし集合）= タグ・ユニーク判定
SADD tags:post:1 "ruby" "redis"
SISMEMBER tags:post:1 "ruby"       # 1（含む）

# Sorted Set（スコア付き集合）= ランキング
ZADD ranking 100 "Alice" 80 "Bob"
ZREVRANGE ranking 0 2 WITHSCORES   # スコア降順 上位3件
```

### ④ アプリのクライアントから直接書く
キャッシュ抽象を介さず、Redisクライアントで素のコマンドを叩く。
```ruby
# Ruby（redis gem）
redis = Redis.new(url: ENV["REDIS_URL"])
redis.set("page_views", 0)
redis.incr("page_views")
redis.setex("session:abc", 3600, "data")   # TTL付き
```
```python
# Python（redis-py）
import redis
r = redis.from_url(os.environ["REDIS_URL"])
r.set("page_views", 0, ex=3600)             # ex=TTL秒
r.incr("page_views")
```
```javascript
// Node.js（ioredis）
const redis = new Redis(process.env.REDIS_URL);
await redis.set("page_views", 0, "EX", 3600);
await redis.incr("page_views");
```

### ⑤ 書いた中身を確認する
```bash
TYPE user:101          # キーの型（string/hash/list/set/zset）
EXISTS user:101        # 1=ある / 0=ない
SCAN 0 MATCH user:* COUNT 100   # キー探索は SCAN を使う（KEYS * は厳禁・後述）
DEL user:101           # 削除
FLUSHDB                # そのDB番号を全消し（本番厳禁）
```

---

## 直接書いたデータは消える？（永続化）
メモリ上なので、再起動で**消えうる**。永続化を有効にすると残せる。
- **RDB（スナップショット）**：一定間隔でメモリ全体をダンプ。再起動で復元できるが、最後のダンプ以降は失われうる。
- **AOF（追記ログ）**：書き込みコマンドを逐次ログ。RDBより失われにくいがファイルは大きい。
- **判断**：キャッシュ用途なら飛んでも作り直せるのでRDBで十分。ジョブやセッションなど飛ぶと困るものはAOFを検討、もしくはRDBに正を置く。

## 実務での勘所
- **TTL設計が肝**。直接書くキーには原則 `EX` を付ける。付け忘れがメモリ枯渇の最大原因。
- **メモリは有限**：`maxmemory` と追い出しポリシー（`allkeys-lru` 等）を設定。上限到達時に何を捨てるかを決めておく。
- **DB番号 or インスタンスで用途を分ける**：キャッシュとジョブキューを同じDBに同居させると競合・誤FLUSHの事故。
- **キー命名規約**：`user:101:name` のように `:` 区切りで名前空間を切ると、SCANや運用がしやすい。
- **マネージド**で運用負荷を下げるのが一般的 → [../インフラ/aws/](../インフラ/aws/)（ElastiCache）/ [../インフラ/gcp/](../インフラ/gcp/)（Memorystore）。

## ハマりどころ / アンチパターン
- **`KEYS *` を本番で実行**：全キー走査で**Redis全体がブロック**（単一スレッドのため他の処理が止まる）。必ず `SCAN` を使う。
- **TTLなしで直接書き続ける**：メモリが無限に増えて `maxmemory` 到達 → 追い出し or 書き込み失敗。
- **巨大な値を1キーに**：数MBの値や巨大Listは単一スレッドを長く占有して全体が遅延。
- **Redis落ち＝アプリ落ち**：キャッシュ用途なら「落ちても素通りで動く」フォールバックにする。正データをRedisだけに置かない。
- **正規化を捨ててRedisに何でも入れる**：構造化された永続データはRDBの仕事。Redisは「速さが要る一時データ」に絞る。
- **`FLUSHDB` / `FLUSHALL` の誤爆**：全消し。本番接続では `--no-auth-warning` 含め慎重に。

## 関連
- Rails での使い方（キャッシュ/Sidekiq/Action Cable/Solid Queueとの関係） → [../バックエンド/ruby/rails/周辺インフラ/redis.md](../バックエンド/ruby/rails/周辺インフラ/redis.md)
- マネージド運用 → [../インフラ/aws/](../インフラ/aws/) / [../インフラ/gcp/](../インフラ/gcp/)
- 永続的なデータは RDB へ → [mysql.md](./mysql.md) / [postgresql.md](./postgresql.md)
