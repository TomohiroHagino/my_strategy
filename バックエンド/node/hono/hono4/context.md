# Context `c`（Hono v4）

## ひとことで言うと
ハンドラ／ミドルウェアに渡ってくる **`c` オブジェクト**。1 リクエストの「**入力（`c.req`）・出力（`c.json` 等）・環境（`c.env`）・受け渡し（`c.set/get`）**」を全部まとめて持つ、Hono の中心。

## 役割・なぜ必要か
- Express の `req` / `res` を **1 つに統合**したもの。これ 1 個でリクエストを読み、レスポンスを作る。
- Web 標準（`Request`/`Response`）の上に薄く乗っており、**ランタイムが違っても同じ `c` の API** で書ける。
- `c.env` で **Workers のバインディング（KV / D1 / R2 など）** にアクセスでき、`c.set/get` で**ミドルウェア → ハンドラ間の値共有**ができる。

## 基本の書き方（コード）
```ts
app.post('/users/:id', async (c) => {
  // ---- 入力：c.req ----
  const id = c.req.param('id')              // パスパラメータ
  const page = c.req.query('page')          // クエリ
  const auth = c.req.header('Authorization')// ヘッダ
  const body = await c.req.json()           // JSONボディ（必ず await）
  // const data = c.req.valid('json')        // バリデータ通過後の値 → validation.md

  // ---- 出力：c.json / c.text / c.html / c.body ----
  c.header('X-Custom', 'value')             // ヘッダ付与
  return c.json({ id, page, body }, 201)    // 第2引数でステータス
})

app.get('/hello', (c) => c.text('hi'))                 // テキスト
app.get('/page',  (c) => c.html('<h1>Hi</h1>'))        // HTML
app.get('/raw',   (c) => c.body('bytes', 200))         // 生ボディ
```

```ts
// ---- 受け渡し：c.set() / c.get() ----
app.use('*', async (c, next) => {
  c.set('requestId', crypto.randomUUID())   // ミドルウェアで詰める
  await next()
})
app.get('/', (c) => c.json({ id: c.get('requestId') })) // ハンドラで取り出す

// ---- 環境：c.env（Workersのバインディング） ----
type Bindings = { MY_KV: KVNamespace; DB: D1Database }
const app = new Hono<{ Bindings: Bindings }>()
app.get('/kv/:key', async (c) => {
  const v = await c.env.MY_KV.get(c.req.param('key'))   // KV を読む
  return c.json({ value: v })
})
```

## 実務での使い方・定番パターン
- **`c.req.json()` は必ず `await`**：戻りは Promise。`const body = await c.req.json()`。
- **`c.set/get` に型を付ける**：`new Hono<{ Variables: { user: User } }>()` とすると `c.get('user')` が型付きになる（認証ミドルウェアでユーザを詰める定番）。→ [auth.md](./auth.md)
- **`c.env` の型は `Bindings` で定義**：`new Hono<{ Bindings: Bindings }>()`。KV / D1 / R2 / Secrets が型安全に。→ [runtimes.md](./runtimes.md)
- **ステータス・ヘッダは出力メソッドの引数 or `c.header()`**：`c.json(data, 201)` / `c.status(204)` / `c.header('Cache-Control', '...')`。
- **`c.req.valid('json')`** でバリデータ済みの安全な値を受け取る（生の `c.req.json()` ではなく）。→ [validation.md](./validation.md)

## ハマりどころ / アンチパターン
- **`c.req.json()` の `await` 忘れ**：Promise をそのまま使い `body.name` が `undefined`。必ず `await`。
- **ボディを二度読む**：`Request` のボディはストリームで**一度しか読めない**。`await c.req.json()` を複数回呼ぶと 2 回目が失敗。読んだ値を変数に保持して使い回す。
- **`c.env` をランタイム非依存と思い込む**：`c.env` は **Workers のバインディング**が中心。Node では存在せず `process.env` を使う。ランタイムで意味が変わる。→ [runtimes.md](./runtimes.md)
- **`c.set/get` をリクエストをまたいで使う**：`c` は**1 リクエスト限り**。グローバル状態の保存場所ではない（共有したいなら外部ストアへ）。
- **`return` を忘れる**：`c.json(...)` を書いても `return` しないと Response が返らず空応答。
- **出力後にヘッダを足す**：`c.json()` を return した後に `c.header()` しても効かない。ヘッダは**返す前**に設定する。

## 関連
[middleware.md](./middleware.md) / [runtimes.md](./runtimes.md) / [validation.md](./validation.md)
