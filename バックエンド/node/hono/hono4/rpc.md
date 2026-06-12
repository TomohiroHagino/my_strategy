# 型安全RPC（hc クライアント）（Hono v4）

## ひとことで言うと
サーバのルート定義から **型を自動共有**し、クライアントで `hc<AppType>()` を使うと `client.users.$get()` のように **リクエスト/レスポンスが型付き**でAPIを呼べる仕組み。OpenAPIやコード生成なしで、TypeScriptの型だけでサーバ↔クライアントを繋ぐ。Honoの目玉機能。

## 役割・なぜ必要か
- フロントとバックでAPIの形（パス・パラメータ・ボディ・レスポンス）が**ズレる事故**を、コンパイル時に止められる。
- `fetch('/api/users')` のような文字列ベースの呼び出しは、パスのtypoやレスポンス型の取り違えを実行時まで気づけない。RPCなら**エディタ補完＋型エラー**で即わかる。
- サーバの型をそのまま `import type` するだけなので、**スキーマの二重管理が不要**。サーバを直せばクライアントの型も自動で追従する。
- モノレポ（フロント＋バック同居）と特に相性が良い。

## 基本の書き方（コード）
サーバ側は **ルートをメソッドチェーンで定義**し、その型を `export type` する。これが最重要ポイント。
```ts
// server.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

const app = new Hono()

// チェーンで繋ぐ → routes に型が溜まる
const routes = app
  .get('/users', (c) => {
    return c.json({ users: [{ id: 1, name: 'taro' }] })
  })
  .get('/users/:id', (c) => {
    const id = c.req.param('id')
    return c.json({ id: Number(id), name: 'taro' })
  })
  .post(
    '/users',
    zValidator('json', z.object({ name: z.string().min(1) })),
    (c) => {
      const { name } = c.req.valid('json') // 型付き
      return c.json({ id: 2, name }, 201)
    }
  )

// この型をクライアントへ渡す（値ではなく型だけexport）
export type AppType = typeof routes
export default app
```

クライアント側は `hc<AppType>()` でクライアントを作る。
```ts
// client.ts
import { hc } from 'hono/client'
import type { AppType } from './server' // 型だけimport

const client = hc<AppType>('http://localhost:8787')

// GET /users → resはレスポンス型を持つ
const res = await client.users.$get()
const data = await res.json() // { users: { id: number; name: string }[] }

// GET /users/:id → paramも型チェックされる
const res2 = await client.users[':id'].$get({ param: { id: '1' } })

// POST /users → jsonボディがzodスキーマ通りでないと型エラー
const res3 = await client.users.$post({ json: { name: 'hanako' } })
const created = await res3.json() // { id: number; name: string }
```

## 実務での使い方・定番パターン
- **zValidatorと併用**：`zValidator('json', schema)` を付けたルートは、クライアントの `$post({ json: ... })` の入力型がスキーマから推論される。入力も出力も型付きになる。→ [validation.md](./validation.md)
- **`InferRequestType` / `InferResponseType`** で型を取り出して再利用：
```ts
import { hc, type InferResponseType, type InferRequestType } from 'hono/client'

const client = hc<AppType>('http://localhost:8787')
const $get = client.users[':id'].$get
type ResUser = InferResponseType<typeof $get>      // レスポンス型
type ReqUser = InferRequestType<typeof $get>['param'] // { id: string }
```
- **モノレポでの型共有**：`packages/server` の `AppType` を `packages/web` から `import type` する。型だけなのでバンドルには含まれず、ビルド境界を越えても軽い。
- **ベースパスの統一**：APIを `/api` 配下に置くなら `app.basePath('/api')` でまとめ、クライアントのURLも合わせる。
- **クエリやヘッダ**も `$get({ query: {...}, header: {...} })` で型付きに渡せる（`zValidator('query', ...)` 等を定義した場合）。

## ハマりどころ / アンチパターン
- **ルートをチェーンしないと型が出ない**（最頻出）。`app.get(...); app.post(...)` と**文ごとに分けて書くと型が蓄積されず**、`client` がただの空オブジェクトになる。必ず `const routes = app.get(...).post(...)` と**1本のチェーン**にする。
- **`export type AppType = typeof routes` を忘れる/値をexportしてしまう**。クライアントが要るのは **値ではなく型**。`export type` で型だけ渡す（バンドル肥大も防げる）。
- **`c.json()` の引数に明示的な型注釈を付けない**こと。型注釈で固定すると推論が崩れることがある。リテラルを返してHonoに推論させるのが基本。
- **`await res.json()` を忘れて `res` を直接使う**。返るのは `Response` なので、データは `.json()`（や `.text()`）で取り出す。ステータス分岐は `if (res.ok)` で。
- **巨大な単一appの型推論が重い**問題。ルートが膨大だとエディタが重くなる。`app.route('/users', usersApp)` で**サブappに分割**し、それぞれをチェーンで組む。
- **バージョン不一致**：サーバとクライアントで `hono` のバージョンがズレると型が噛み合わない。モノレポなら同一バージョンに揃える。

## 関連: [routing.md](./routing.md) / [validation.md](./validation.md)
