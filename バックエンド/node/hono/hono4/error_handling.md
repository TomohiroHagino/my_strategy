# エラー処理（Error Handling）（Hono v4）

## ひとことで言うと
ハンドラやミドルウェア内で **`throw` された例外を `app.onError((err, c) => ...)` で集約処理**する仕組み。意図したHTTPエラーは **`HTTPException`**（`import { HTTPException } from 'hono/http-exception'`）を `throw` し、未定義ルートは **`app.notFound()`** で受ける。Expressの「引数4個のエラーミドルウェア」に相当するが、Honoは `throw` → `onError` の流れがシンプル。

## 役割・なぜ必要か
- エラー応答の整形（ステータスコード・JSON形式・ログ）を**1か所（`onError`）に集約**でき、各ハンドラは「正常系を書いて、異常時は `throw` するだけ」で済む。
- `HTTPException` を使えば、ハンドラの奥深くからでも「401で止める」「403で止める」を**例外1行**で表現でき、途中の `if` ネストを減らせる。
- ミドルウェアで起きた例外も含め、アプリ全体のエラーが `onError` に集まるので、**本番でのエラー漏洩防止**やログ出力を統一管理できる。

## 基本の書き方（コード）
```ts
import { Hono } from 'hono'
import { HTTPException } from 'hono/http-exception'  // ← import 元に注意

const app = new Hono()

// 1) 意図したエラーは HTTPException を throw（ステータス＋メッセージ）
app.get('/secret', (c) => {
  const token = c.req.header('Authorization')
  if (!token) {
    throw new HTTPException(401, { message: '認証が必要です' }) // ← onError へ飛ぶ
  }
  return c.json({ ok: true })
})

// 想定外のエラー（バグ・DB障害など）も throw すれば onError が拾う
app.get('/boom', () => {
  throw new Error('予期しない内部エラー')
})

// 2) onError：全エラーの集約点（ハンドラ／ミドルウェアの throw が来る）
app.onError((err, c) => {
  if (err instanceof HTTPException) {
    // HTTPException は自前のレスポンスを持つ（getResponse）か、整形して返す
    return c.json({ error: err.message }, err.status)
  }
  // 想定外エラー：本番では詳細を返さない（情報漏洩防止）
  console.error(err)                       // ログには詳細を残す
  return c.json({ error: 'Internal Server Error' }, 500)
})

// 3) notFound：未定義ルート（404）の整形
app.notFound((c) => {
  return c.json({ error: 'Not Found', path: c.req.path }, 404)
})

export default app
```

## 実務での使い方・定番パターン
- **意図したエラーは `HTTPException`、想定外は素の `throw`**：認証・権限・バリデーション等の「業務上のエラー」は `HTTPException(4xx, {message})`。バグやDB障害は `throw new Error(...)` で構わない（どちらも `onError` が拾う）。
- **`onError` で分岐**：`err instanceof HTTPException` なら `err.status` / `err.message` を使って整形、それ以外は500に丸める。`HTTPException` は `err.getResponse()` で元のレスポンスをそのまま返すこともできる。
- **原因（cause）を添える**：`new HTTPException(502, { message: 'upstream失敗', cause: e })` のように `cause` を渡すとログで追跡しやすい。
- **共通エラーレスポンス形式に揃える**：`{ ok: false, error: ... }` のようなアプリ共通の封筒形式を `onError` / `notFound` で統一する。
- **ミドルウェアの例外も集約される**：認証ミドルウェア内の `throw new HTTPException(401, ...)` も `onError` に届くので、認証失敗の整形も1か所で済む（→ [middleware.md](./middleware.md)）。

```ts
import { bearerAuth } from 'hono/bearer-auth'

// ミドルウェアで弾いた認証エラーも onError が整形する
app.use('/api/*', bearerAuth({ token: process.env.API_TOKEN! }))
// → トークン不一致時は内部で HTTPException(401) が throw され onError へ
```

## ハマりどころ / アンチパターン
- **`HTTPException` の import 元間違い**：正しくは **`import { HTTPException } from 'hono/http-exception'`**。`hono` 本体や別パスから import しようとして見つからず詰まる事故が多い。
- **本番でエラー詳細をそのまま返す**：`return c.json({ error: err.message }, 500)` を想定外エラーにも適用すると、スタックや内部情報が**クライアントに漏れる**。想定外（非 `HTTPException`）は固定文言の500に丸め、詳細は `console.error` でログにのみ残す。
- **`onError` を定義し忘れる**：未定義だとHonoのデフォルト500応答になり、整形やログが効かない。アプリ起動時に必ず `app.onError` / `app.notFound` を登録する。
- **`throw` せず握りつぶす**：`try/catch` で握りつぶして何も返さないと、`onError` に届かず空レスポンスになる。捕捉したら整形して返すか、再 `throw` して `onError` に委ねる。
- **`notFound` と `onError` の混同**：ルート未マッチは `notFound`、例外（throw）は `onError`。404を `onError` で扱おうとしてもそこには来ない。

## 関連
[middleware.md](./middleware.md) / [context.md](./context.md)
