# ルーティング（Routing）（Hono v4）

## ひとことで言うと
「**どのメソッド・どのパスのリクエストを、どのハンドラで処理するか**」の対応づけ。`app.get('/path', handler)` のように書く。

## 役割・なぜ必要か
- HTTP の入口を整理する基本。`GET /users/:id` のような URL を、対応する関数へ振り分ける。
- Hono のルーティングは**高速**（複数ルータ実装を内部で使い分け）で、書き味も Express 系に近く学習コストが低い。
- **型安全 RPC を使うなら、ルート定義の書き方（チェーン）が型生成に直結する**。ここを最初に押さえると後で効く。→ [rpc.md](./rpc.md)

## 基本の書き方（コード）
```ts
import { Hono } from 'hono'

const app = new Hono()

// メソッドごとのハンドラ
app.get('/users', (c) => c.json([{ id: 1 }]))
app.post('/users', (c) => c.json({ created: true }, 201))
app.put('/users/:id', (c) => c.json({ updated: c.req.param('id') }))
app.delete('/users/:id', (c) => c.json({ deleted: c.req.param('id') }))

// パスパラメータ → c.req.param()
app.get('/users/:id', (c) => {
  const id = c.req.param('id')         // 単一
  // const { id } = c.req.param()      // まとめて取得も可
  return c.json({ id })
})

// クエリ文字列 → c.req.query()
app.get('/search', (c) => {
  const q = c.req.query('q')           // ?q=hono の値
  const all = c.req.query()            // 全部オブジェクトで
  return c.json({ q, all })
})

// ワイルドカード
app.get('/files/*', (c) => c.text('any file path'))
```

```ts
// グループ化：サブアプリを app.route() でマウント
const api = new Hono()
api.get('/posts', (c) => c.json(['post1']))   // 実際は /api/posts
api.get('/posts/:id', (c) => c.json({ id: c.req.param('id') }))

const app = new Hono()
app.route('/api', api)                         // /api 以下にまとめる
```

## 実務での使い方・定番パターン
- **機能ごとにサブアプリへ分割**し `app.route('/api/users', usersApp)` でマウント。ファイルが肥大化せず見通しが良い。
- **RPC を使うならメソッドチェーンで定義する**：
  ```ts
  const route = app
    .get('/users', (c) => c.json([]))
    .post('/users', (c) => c.json({}, 201))
  export type AppType = typeof route   // ← この型をクライアントへ渡す
  ```
  チェーンしないと `AppType` に全ルートが乗らず、RPC クライアントの型が欠ける。→ [rpc.md](./rpc.md)
- **共通ミドルウェアはパス単位で**：`app.use('/api/*', cors())` のように前置きで適用。→ [middleware.md](./middleware.md)
- **パラメータの正規表現**：`/:id{[0-9]+}` で数値のみ受ける等、ルートで制約できる。
- **`app.on(['GET','POST'], path, handler)`** で複数メソッドをまとめられる。

## ハマりどころ / アンチパターン
- **ルート順（登録順）の取り違え**：より具体的なルートを先に。`/users/me` を `/users/:id` より**後**に書くと `:id='me'` に飲まれる。
- **`c.req.param('id')` を `c.req.params` と書く**：Hono は `param()`（関数）。Express の `req.params`（プロパティ）と違う。
- **クエリの `c.req.query()` で配列が取れない**：同名複数キーは `c.req.queries('key')`（複数形）を使う。
- **RPC を使うのにルートを `app.get(...)` と**バラバラに書く**：型が連結されない。チェーンか、サブアプリを `app.route()` で合成する。
- **ワイルドカード `*` の置き場所**：先に広いワイルドカードを置くと後続の具体ルートに届かない。具体 → 広い の順で。
- **`app.route()` のマウントパスを二重に書く**：サブアプリ側で `'/api/posts'` と書き、さらに `app.route('/api', ...)` すると `/api/api/posts` になる。サブアプリ側は `/posts` だけにする。

## 関連
[context.md](./context.md) / [rpc.md](./rpc.md) / [middleware.md](./middleware.md)
