# マルチランタイム（Workers / Deno / Bun / Node）（Hono v4）

## ひとことで言うと
**同じアプリのコードを、複数の実行環境（ランタイム）でそのまま動かせる**仕組み。Honoは `Request`/`Response` というWeb標準を土台にしているので、Cloudflare Workers / Node / Bun / Deno / Vercel / Lambda の違いを **アダプタ（エントリ部分）だけで吸収**できる。Honoの目玉。

## 役割・なぜ必要か
- 環境ごとに別フレームワークを覚え直さなくていい。**ルートやミドルウェアの書き方は共通**で、起動の入口だけ差し替える。
- 開発はNode/Bun、本番はCloudflare Workers、といった**移行・併用**がやりやすい。
- Honoのアプリ本体（`const app = new Hono()`）はどこでも同じ。違うのは「**誰がリクエストをappに渡すか**」だけ。
- `c.env` でランタイム固有のリソース（Workersのバインディング等）にアクセスできる。ただし中身はランタイムで異なる。

## 基本の書き方（コード）
アプリ本体は共通。エントリだけランタイムごとに変える。
```ts
// app.ts（共通：どのランタイムでも同じ）
import { Hono } from 'hono'

const app = new Hono()
app.get('/', (c) => c.text('Hello Hono'))

export default app
```

```ts
// Cloudflare Workers：app を default export するだけ
// wrangler.toml で main = "src/index.ts"
import app from './app'
export default app // Workersが fetch ハンドラとして拾う
```

```ts
// Node：@hono/node-server の serve でラップ
import { serve } from '@hono/node-server'
import app from './app'

serve({ fetch: app.fetch, port: 3000 }, (info) => {
  console.log(`Listening on http://localhost:${info.port}`)
})
```

```ts
// Bun：default export（Bun.serve が fetch を拾う）
import app from './app'
export default app
// もしくは: export default { port: 3000, fetch: app.fetch }
```

```ts
// Deno：Deno.serve に fetch を渡す
import app from './app'
Deno.serve(app.fetch)
```

## 実務での使い方・定番パターン
- **Cloudflare Workers のバインディング**は `c.env` から取る（KV / D1 / R2 / Queues など）。型は `Hono<{ Bindings: Env }>` で付ける：
```ts
type Bindings = { MY_KV: KVNamespace; DB: D1Database }
const app = new Hono<{ Bindings: Bindings }>()

app.get('/cache/:key', async (c) => {
  const v = await c.env.MY_KV.get(c.req.param('key')) // 型付き
  return c.json({ value: v })
})
```
- **環境変数の取り方が違う**点に注意。Workersは `c.env.SECRET`、Nodeは `process.env.SECRET`。共通化したいなら自前のヘルパーで吸収する。→ [context.md](./context.md)
- **静的ファイル配信**もアダプタ別：Nodeは `@hono/node-server/serve-static`、Workersは Workers Assets / `serveStatic`、Bun/Denoはそれぞれの `serveStatic`。
- **デプロイ単位**：Workersは `wrangler deploy`、Nodeはプロセス常駐（PM2やコンテナ）、Bun/Denoは各ランタイムの起動コマンド。
- **ローカル開発**：Workersは `wrangler dev`（`c.env` のバインディングをエミュレート）、Nodeは `tsx watch src/index.ts` などでホットリロード。

```bash
# 起動例
npx wrangler dev          # Cloudflare Workers（ローカルエミュレート）
npx tsx watch src/node.ts # Node（@hono/node-server）
bun run src/bun.ts        # Bun
deno run --allow-net src/deno.ts  # Deno
```

## ハマりどころ / アンチパターン
- **ランタイムごとのエントリ/環境変数/バインディングを取り違える**（最頻出）。Workersに `process.env` を書いても動かない（`c.env` を使う）。Nodeに `serve()` を忘れると起動しない。
- **Node固有APIを使うと移植性が落ちる**。`fs`・`path`・`Buffer`・`process` などを本体ロジックに直書きすると、WorkersやDenoで動かなくなる。**Web標準（`fetch`/`Request`/`Response`/`crypto.subtle`/`URL`）で書く**のが鉄則。
- **Workersの `c.env` はリクエストごとに渡される**。モジュールのトップレベルで `c.env` 相当を参照しようとすると取れない。ハンドラ内で `c.env` から取る。
- **`@hono/node-server` の入れ忘れ**。Nodeで `serve` を使うには別パッケージが要る（`npm i @hono/node-server`）。Honoコアには含まれない。
- **CPU/メモリ・実行時間の制約差**。Workersは実行時間やバンドルサイズに制限があり、重い依存（巨大npmやNodeネイティブモジュール）はそのまま載らないことがある。
- **動的import/eval系**はランタイムによって挙動・許可が異なる。Workersは特に制約が厳しい。

## 関連: [getting_started.md](./getting_started.md) / [context.md](./context.md)
