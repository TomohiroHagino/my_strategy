# バリデーション（Validation）（Hono v4）

## ひとことで言うと
**`@hono/zod-validator` の `zValidator()` をルートに挟む**ことで、リクエストの入力（JSON/クエリ/パスパラメータ/フォーム）を**Zodスキーマで検証**し、ハンドラ内で **`c.req.valid('json')` から型付きの検証済みデータ**を受け取る仕組み。検証失敗時は**自動で400**を返す。Zodで「型定義」と「実行時検証」を一本化できるのが要点。

## 役割・なぜ必要か
- 外部から来る入力（ボディ・クエリ）は**信用できない**。処理前に必ず検証する必要がある。
- 手書きの `if (!body.name) ...` を散らかす代わりに、**スキーマ1つで検証＋型付け**を完結。検証を通った後のデータは TypeScript の型が付くので、`c.req.valid('json').name` が安全に書ける。
- 検証ロジックがルート定義に**宣言的に乗る**ため、RPC（`hc`）で型が伝播し、クライアント側も入力型を共有できる（→ [rpc.md](./rpc.md)）。

## 基本の書き方（コード）
```ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'  // npm i @hono/zod-validator zod
import { z } from 'zod'

const app = new Hono()

// 1) スキーマ定義（型と検証ルールを一本化）
const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().int().min(0).optional(),
})

// 2) ルートに zValidator を挟む（'json' = リクエストボディを検証）
app.post(
  '/users',
  zValidator('json', createUserSchema),  // ← 検証失敗なら自動で 400
  (c) => {
    // 3) 検証済みデータは c.req.valid('json') から取る（型付き！）
    const body = c.req.valid('json')      // { name: string; email: string; age?: number }
    return c.json({ created: body.name }, 201)
  }
)

// クエリ文字列の検証（'query'）
const listSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),  // coerce で文字列→数値
  limit: z.coerce.number().int().max(100).default(20),
})
app.get('/users', zValidator('query', listSchema), (c) => {
  const { page, limit } = c.req.valid('query')  // 型付きの検証済みクエリ
  return c.json({ page, limit })
})

// パスパラメータの検証（'param'）
app.get(
  '/users/:id',
  zValidator('param', z.object({ id: z.string().uuid() })),
  (c) => {
    const { id } = c.req.valid('param')
    return c.json({ id })
  }
)

export default app
```

## 実務での使い方・定番パターン
- **ターゲット種別を使い分ける**：`zValidator(target, schema)` の `target` は
  - `'json'` … `Content-Type: application/json` のボディ。
  - `'query'` … クエリ文字列（`?page=2`）。値は文字列なので `z.coerce.number()` で変換。
  - `'param'` … パスパラメータ（`/users/:id` の `id`）。
  - `'form'` … `multipart/form-data` / `x-www-form-urlencoded` のフォーム送信。
  - `'header'` / `'cookie'` … ヘッダ・クッキーの検証も可能。
- **複数同時に挟める**：1ルートに `zValidator('param', ...)` と `zValidator('json', ...)` を両方挟み、それぞれ `c.req.valid('param')` / `c.req.valid('json')` で取得。
- **スキーマは別ファイルに切り出す**：`schemas/user.ts` などにまとめ、ルートとクライアント（RPC）で共有。Zodの `z.infer<typeof schema>` で型も再利用できる。
- **エラー整形をカスタムしたい時**は `zValidator('json', schema, (result, c) => { if (!result.success) return c.json(...) })` の第3引数フックで上書きする。

```ts
// エラーレスポンスをアプリ共通形式に揃える例
app.post(
  '/items',
  zValidator('json', createUserSchema, (result, c) => {
    if (!result.success) {
      return c.json({ ok: false, errors: result.error.flatten() }, 400)
    }
  }),
  (c) => c.json({ ok: true })
)
```

## ハマりどころ / アンチパターン
- **検証済みデータを `c.req.json()` から取ってしまう**：`c.req.json()` は**生データ**（未検証・型なし）。検証済み・型付きで欲しいなら必ず **`c.req.valid('json')`** から取る。ここを取り違えると型安全のメリットが消える。
- **`target` の指定ミス**：`'query'` を検証したのに `c.req.valid('json')` を呼ぶと型が合わない／取れない。挟んだ `zValidator` の `target` と `valid()` の引数を一致させる。
- **クエリを `z.number()` で直に検証**：クエリ値は常に**文字列**。`z.number()` だと弾かれるので `z.coerce.number()` を使う。
- **スキーマ未定義で `min()` 等のルール漏れ**：`z.string()` だけでは空文字も通る。`.min(1)` や `.email()` など制約を明示しないと「検証している気になる」だけ。
- **`@hono/zod-validator` と `zod` の入れ忘れ**：両方 `npm i` が必要。`zod` 本体がないと動かない。

## 関連
[context.md](./context.md) / [rpc.md](./rpc.md)
