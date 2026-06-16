# DB（Database）（Hono v4）

## ひとことで言うと
Honoには**DB機能は内蔵されていない**。Honoはルーティング＋Web標準のフレームワークで、永続化は**自分でORM/ドライバを選ぶ**。実務の定番は **Drizzle ORM**（軽量・型安全・Edge/D1向き）か **Prisma**。データ自体は **Cloudflare D1**（Workers上のSQLite）/ PostgreSQL / MySQL 等を使う。

## 役割・なぜ必要か
- Honoはどのランタイム（Workers / Deno / Bun / Node）でも動くので、**DB接続もランタイムに依存する**。Node前提の重いドライバはEdgeで動かない。
- だからこそ「Edge対応のORM/ドライバを選ぶ」判断が要る。Drizzleは依存が少なく、Workers/D1とも相性が良いため、Hono界隈では人気。
- D1を使うと、DB接続が `c.env.DB`（Workersのバインディング）としてContext経由で渡ってくる。接続文字列を環境変数で持つ従来型とは流儀が違う。

## 基本の書き方（コード）
```ts
// Cloudflare D1 + Drizzle（Workers）
import { Hono } from 'hono'
import { drizzle } from 'drizzle-orm/d1'
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'

// スキーマ定義（型がここから生える）
export const users = sqliteTable('users', {
  id: integer('id').primaryKey(),
  email: text('email').notNull(),
  name: text('name'),
})

// c.env の型（D1Database はWorkersのバインディング）
type Bindings = { DB: D1Database }
const app = new Hono<{ Bindings: Bindings }>()

app.get('/users', async (c) => {
  // c.env.DB（D1）からdrizzleインスタンスを毎リクエスト生成するのが定石
  const db = drizzle(c.env.DB)
  const rows = await db.select().from(users).all()
  return c.json(rows)
})

app.post('/users', async (c) => {
  const body = await c.req.json<{ email: string; name?: string }>()
  const db = drizzle(c.env.DB)
  const inserted = await db.insert(users).values(body).returning()
  return c.json(inserted[0], 201)
})

export default app
```
```bash
# wrangler.toml に D1 バインディングを定義（抜粋）
# [[d1_databases]]
# binding = "DB"          # → c.env.DB でアクセス
# database_name = "my-db"
# database_id = "xxxx-xxxx"

# マイグレーション（drizzle-kit）
npx drizzle-kit generate         # スキーマからSQL生成
npx wrangler d1 migrations apply my-db --local   # D1へ適用
```
```ts
// Prisma を使う場合（接続文字列はランタイムの環境変数から）
import { PrismaClient } from '@prisma/client'
// Edge では @prisma/client/edge + Accelerate/Data Proxy が必要な点に注意
const prisma = new PrismaClient()
app.get('/posts', async (c) => c.json(await prisma.post.findMany()))
```

## 実務での使い方・定番パターン
- **Drizzle + D1（Workers）**：EdgeでサーバレスにSQLを動かす王道。`drizzle(c.env.DB)` を**ハンドラ内で毎回生成**する（Workersはリクエストごとに隔離されるため、グローバル接続を持ち回らない）。
- **接続はランタイム依存**：Node なら `pg` / `mysql2` + Drizzle/Prisma、Workers なら D1 や Hyperdrive、Deno/Bun はそれぞれのドライバ。**同じHonoコードでもDB層だけは環境ごとに差し替える**設計にする。
- **型はスキーマ起点**：DrizzleもPrismaもスキーマ定義から型が生え、`c.req.valid()`（zod-validator）と合わせれば入力→DBまで型で繋がる。
- **環境変数 / バインディングの注入**：D1は `c.env.DB`、接続文字列は `c.env.DATABASE_URL` のように **Context経由**で受け取り、`process.env` 直参照は避ける（移植性のため）。
- **マイグレーションはツールで**：drizzle-kit / prisma migrate / wrangler d1 migrations を使い、手書きSQLの当て込みを避ける。

## N+1 と Edge 特有の現象・対策
Edgeランタイム（Workers等）は**1リクエストあたりのクエリ回数・実行時間に上限**があり、N+1は通常のサーバより致命的になりやすい。
| 現象 | なぜ起きる | 対策 |
|---|---|---|
| **N+1** | ループ内で関連を都度引く | Drizzle の **Relational Queries** `db.query.users.findMany({ with: { posts: true } })` で関連をまとめ取り。生クエリは JOIN か **`inArray(col, ids)`** でIN取得 |
| **D1のクエリ回数/サブリクエスト上限超過** | Workersは1リクエストで実行できるD1クエリ数・CPU時間に制限。N+1や大量クエリで上限に達する | クエリ数を減らす（まとめ取り）。複数の書き込みは **`db.batch([...])`**（D1）で1往復に集約 |
| **リクエスト毎インスタンス必須** | Workersはリクエストごとに隔離。グローバルな接続プールを持てない | `drizzle(c.env.DB)` は**ハンドラ内で生成**（モジュールトップ生成は不可） |
| **Edge非対応ドライバ** | TCPプール前提の生 `pg` 等はEdgeで動かない/枯渇 | **HTTPベース**を使う（D1 / Hyperdrive / Neon serverless / Prisma Accelerate） |
| **長時間トランザクション** | D1等はロック保持・長時間Txに不向き | トランザクションは短く。原子性は `db.batch()` で確保 |
| 繰り返しクエリのオーバーヘッド | 毎回プランを組む | Drizzle の **`.prepare()`** でプリペアド化 |
| 全件取得 | ページング無しで重い | `limit`/`offset` か cursor。スキーマに `index()` を張る |

## ハマりどころ / アンチパターン
- **「Honoに付いている」前提**：DB機能は内蔵されていない。ORM・ドライバ・マイグレーションは**全部自前で選定**する。
- **Edge非対応のドライバを使う**：従来のNode用ドライバ（TCPコネクションプール前提の `pg` 生接続等）はWorkers等のEdgeで動かない／不安定。**Edge対応か**を必ず確認（D1 / Hyperdrive / Prisma Accelerate / Neon serverless 等）。
- **コネクションプール問題**：Edgeは多数の隔離環境が短命に動くため、従来型プールが枯渇しやすい。Edgeではサーバレス向け接続（HTTPベース / プロキシ）を選ぶ。
- **`c.env.DB` を `process.env` で取りに行く**：D1のバインディングは `c.env` 経由でしか来ない。Workersに `process.env` は無い前提で書く。
- **drizzleインスタンスをモジュールトップで生成**：Workersでは `c.env` が無い文脈で `c.env.DB` を参照できない。**ハンドラ内生成**にする。
- **ローカルと本番でマイグレーション差**：`--local` と本番D1は別物。両方に `migrations apply` する手順を漏らさない。

## 関連
[context.md](./context.md) / [runtimes.md](./runtimes.md)
