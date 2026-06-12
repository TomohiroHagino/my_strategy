# 認証（jwt / bearer ミドルウェア）（Hono v4）

## ひとことで言うと
「このリクエストは誰か」「アクセスして良いか」を確かめる仕組み。Honoは **組込ミドルウェア**として `jwt`（JWTトークン検証）・`bearerAuth`（固定トークン）・`basicAuth`（ID/パスワード）を持ち、ミドルウェアを通すだけで保護できる。検証済みの情報は **`c.get(...)`** で後続ハンドラに渡せる。

## 役割・なぜ必要か
- 認証ロジックを各ハンドラに書くと散らかる。**ミドルウェアに寄せれば、保護したいルート群にまとめて適用**できる（onionモデルで先に走る）。→ [middleware.md](./middleware.md)
- `jwt` ミドルウェアは「ヘッダの `Authorization: Bearer xxx` を取り出し、署名を検証し、失敗なら401を返す」までを自動でやる。成功時はペイロードを `c.get('jwtPayload')` で取れる。
- 認証（誰か）と**認可（何をして良いか）**は別。ミドルウェアは認証まで。認可はペイロードの中身（ロール等）を見て自分で判断する。

## 基本の書き方（コード）
```ts
import { Hono } from 'hono'
import { jwt, sign } from 'hono/jwt'
import { bearerAuth } from 'hono/bearer-auth'
import { basicAuth } from 'hono/basic-auth'

const app = new Hono()

// 1) JWT：/api/* を保護。secretで署名検証
app.use('/api/*', jwt({ secret: 'my-secret' }))

app.get('/api/me', (c) => {
  const payload = c.get('jwtPayload') // 検証済みペイロード
  return c.json({ userId: payload.sub })
})

// トークン発行（ログイン成功後など）
app.post('/login', async (c) => {
  const payload = {
    sub: 'user-123',
    role: 'admin',
    exp: Math.floor(Date.now() / 1000) + 60 * 60, // 1時間後に期限切れ
  }
  const token = await sign(payload, 'my-secret')
  return c.json({ token })
})

// 2) bearerAuth：固定トークン（管理API等のシンプル保護）
app.use('/admin/*', bearerAuth({ token: 'static-admin-token' }))

// 3) basicAuth：ID/パスワード
app.use('/private/*', basicAuth({ username: 'user', password: 'pw' }))
```

カスタム認証は自前ミドルウェアで `c.set` → 後続で `c.get`。
```ts
import { createMiddleware } from 'hono/factory'

// 型を付けると c.get('user') が型付きになる
type Env = { Variables: { user: { id: string; role: string } } }

const authMiddleware = createMiddleware<Env>(async (c, next) => {
  const token = c.req.header('Authorization')?.replace('Bearer ', '')
  if (!token) return c.json({ error: 'unauthorized' }, 401)

  const user = await lookupUser(token) // DB等で検証
  if (!user) return c.json({ error: 'unauthorized' }, 401)

  c.set('user', user) // 後続ハンドラへ受け渡し
  await next()
})

const app2 = new Hono<Env>()
app2.use('/app/*', authMiddleware)
app2.get('/app/profile', (c) => {
  const user = c.get('user') // 型付きで取れる
  return c.json({ id: user.id })
})
```

## 実務での使い方・定番パターン
- **secretは `c.env` から取る**（特にWorkers）。ミドルウェアは関数で渡せるので、リクエストごとに環境から読める：
```ts
app.use('/api/*', (c, next) =>
  jwt({ secret: c.env.JWT_SECRET })(c, next)
)
```
- **認可（ロールチェック）は別ミドルウェア**に切り出す。`jwt` の後段で `c.get('jwtPayload').role` を見て、足りなければ403を返す。
- **トークン期限**は `exp`（UNIX秒）で必ず付ける。`jwt` ミドルウェアは期限切れを自動で弾く。
- **リフレッシュトークン**はアクセストークンと別に管理し、漏洩時の影響を絞る。
- **ログインの入力検証**は zValidator と組む（メール形式・必須など）。→ [validation.md](./validation.md)
- 認証失敗は `HTTPException` に寄せて `onError` で一元整形してもよい。→ [error_handling.md](./error_handling.md)

```bash
# 動作確認：トークンを付けてアクセス
curl http://localhost:3000/api/me \
  -H "Authorization: Bearer <token>"
```

## ハマりどころ / アンチパターン
- **secretをコードに直書き**（最頻出かつ重大）。`jwt({ secret: 'my-secret' })` をそのまま本番に出すのはNG。**環境変数 / `c.env` / シークレットマネージャ**から取る。漏れたら即ローテーション。
- **認証だけして認可を忘れる**。JWTが正しい＝ログイン済みなだけで、「adminか」は別。ペイロードのロールを**自分で検証**しないと権限昇格を許す。
- **`exp` を付けない**と期限なしトークンになり、漏洩時に無限に有効。発行時に必ず期限を入れる。
- **`c.set` した値を `c.get` で取れない**：型 `Variables` を `Hono<Env>` に渡し、`c.set` と同じキーで取る。ミドルウェアの**適用順/パス**がズレてset前に取ると `undefined`。
- **ペイロードを信用しすぎる**。JWTは「署名済み＝改ざんなし」を保証するだけで、中身が最新の権限とは限らない。失効（ログアウト・BAN）はサーバ側の確認やトークン失効リストで担保する。
- **basicAuthを平文HTTPで使う**とID/PWが丸見え。必ずHTTPS下で。

## 関連: [middleware.md](./middleware.md) / [context.md](./context.md)
