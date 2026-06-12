# ミドルウェア（Middleware）（Hono v4）

## ひとことで言うと
**`async (c, next) => { /*前*/ await next(); /*後*/ }` という形の関数**で、リクエストとハンドラの「間」に挟まる部品。`app.use()` で登録すると、リクエストはこの関数の列を**onion（玉ねぎ）状**に通り抜ける。`await next()` を境に「前処理」と「後処理」が分かれるのがExpressとの最大の違い＝**Honoミドルウェアの核**。

## 役割・なぜ必要か
- ログ・認証・CORS・キャッシュ・セキュリティヘッダなど、**全ルート（または一部）に共通する前後処理**を1か所にまとめられる。
- `await next()` の**前**＝リクエストが奥へ進む前の処理（認証チェック・計測開始）、**後**＝レスポンスが返ってきた後の処理（ヘッダ追加・ログ出力・計測終了）。1つの関数で「入口」と「出口」の両方を書ける。
- Expressは「`next()` で次へ渡して終わり（一方通行）」だが、Honoは `await next()` で**奥の処理が完了するまで待ち、戻ってくる**。これがonionモデル。

## 基本の書き方（コード）
```ts
import { Hono } from 'hono'
import { logger } from 'hono/logger'
import { cors } from 'hono/cors'

const app = new Hono()

// 1) グローバル：全リクエストが通る（'*' は全パス）
app.use('*', logger())          // 組込ミドルウェア（アクセスログ）
app.use('*', cors())            // CORSヘッダ付与（別オリジン許可）

// 2) カスタムミドルウェア：onionモデルを体感する
app.use('*', async (c, next) => {
  const start = Date.now()      // ── 前処理（await next() より前）──
  await next()                  // ← ここで奥のミドルウェア／ハンドラへ進む
  const ms = Date.now() - start // ── 後処理（戻ってきた後）──
  c.header('X-Response-Time', `${ms}ms`)  // レスポンスにヘッダ追加
})

// 3) パス指定：特定パス配下だけに適用
const requireAuth = async (c: any, next: any) => {
  if (!c.req.header('Authorization')) {
    return c.json({ error: '認証が必要です' }, 401) // ここで終了（next 呼ばない）
  }
  await next()                  // OK なら次へ
}
app.use('/admin/*', requireAuth)  // /admin 以下は全て認証

app.get('/me', (c) => c.json({ user: 'alice' }))

export default app
```

## 実務での使い方・定番パターン
- **パスパターンで適用範囲を決める**
  - `app.use('*', mw)` … 全リクエスト共通（ログ・CORS・セキュリティヘッダ）。
  - `app.use('/admin/*', mw)` … 特定パス配下だけ（`/admin/users` なども含む）。
  - `app.get('/x', mw, handler)` … ルート単位。`get` の引数に複数並べると左から実行。
- **組込ミドルウェア**（`hono/...` から import、`npm i` 不要で同梱）
  - `import { cors } from 'hono/cors'` … CORSヘッダ付与。
  - `import { logger } from 'hono/logger'` … アクセスログ整形。
  - `import { jwt } from 'hono/jwt'` … JWT検証（→ [auth.md](./auth.md)）。
  - `import { cache } from 'hono/cache'` … レスポンスキャッシュ（Workers等のCache API利用）。
  - `import { secureHeaders } from 'hono/secure-headers'` … セキュリティ用HTTPヘッダ（helmet相当）。
  - `import { basicAuth } from 'hono/basic-auth'` … Basic認証。
  - `import { bearerAuth } from 'hono/bearer-auth'` … Bearerトークン認証。
  - `import { etag } from 'hono/etag'` … ETag付与（条件付きGET）。
  - `import { compress } from 'hono/compress'` … レスポンス圧縮（gzip等）。
- **適用順は登録順**：`secureHeaders` / `cors` は早い段階で、`logger` は計測したいので最前段に置くことが多い。認証は保護対象ルートの「前」に。

```ts
import { secureHeaders } from 'hono/secure-headers'
import { compress } from 'hono/compress'
import { etag } from 'hono/etag'

app.use('*', logger())          // 計測のため最前段
app.use('*', secureHeaders())   // セキュリティヘッダ（早めに）
app.use('*', cors())            // CORS（ルートより前）
app.use('*', etag())            // ETag付与
app.use('*', compress())        // 圧縮（出口側で効く）
// …この後にルート定義…
```

## ハマりどころ / アンチパターン
- **`await next()` 忘れ**：最頻出。`await next()` を書かないと**奥のハンドラへ進まず**、後続が実行されない（多くは空レスポンスや404相当に）。前処理だけで終わるミドルウェアでも、通す気なら必ず `await next()` を呼ぶ。
- **`next()` の `await` 漏れ**：`next()` を `await` せずに後処理を書くと、後処理が**ハンドラ完了前に走ってしまう**（onionが崩れる）。後処理でレスポンスを触る場合は必ず `await`。
- **適用順ミス**：認証ミドルウェアを保護対象ルートより「後」に登録すると、認証が効かない。`secureHeaders` を `cors` の後に置くなど、順序で挙動が変わる。
- **パスパターン誤解**：`app.use('/users', mw)` は**そのパスだけ**にしかマッチしない。配下も含めたいなら `app.use('/users/*', mw)` とワイルドカードを付ける（Expressの前方一致とは異なる）。
- **`return c.json(...)` で止めるか `await next()` で通すか**：認証NGなどで止めたい時は `return c.json(..., 401)` のように**応答を返して `next` を呼ばない**。通したい時は `await next()`。どちらか必ず一方。

## 関連
[context.md](./context.md) / [error_handling.md](./error_handling.md)
