# サーバAPI（Nitro / server/api）（Nuxt 3）

## ひとことで言うと
`server/` ディレクトリに置いたファイルが、そのまま **APIエンドポイント（やサーバルート）になる仕組み**。中身は Nuxt 3 に内蔵された **Nitro サーバエンジン**が動かしており、フロントと同じプロジェクトにバックエンドAPIを同居（フルスタック）できる。

## 役割・なぜ必要か
- 「ユーザー一覧を返すAPIが欲しい」というとき、別の Express / Rails を立てなくても、`server/api/users.get.ts` を置くだけで `/api/users` が生える。
- ここは **サーバ側だけで実行される**ので、DB接続文字列・APIシークレット・トークンを安全に扱える（ブラウザへは配信されない）。クライアントの `useFetch` からこの内部APIを叩く、という構成が定番。
- Nitro はデプロイ先非依存（Node・Cloudflare・Vercel・Edge…）なので、同じコードを各種 preset でどこへでも出せる。→ [deployment.md](./deployment.md)

## 基本の書き方（コード）
```ts
// server/api/users.get.ts  →  GET /api/users
export default defineEventHandler(async (event) => {
  // クエリ取得： /api/users?limit=10
  const { limit } = getQuery(event)
  const users = await fetchUsersFromDb(Number(limit ?? 20))
  return users // 戻り値が自動でJSON化される
})
```

```ts
// server/api/users.post.ts  →  POST /api/users
export default defineEventHandler(async (event) => {
  const body = await readBody(event) // リクエストボディを読む
  if (!body?.name) {
    throw createError({ statusCode: 400, statusMessage: 'name required' })
  }
  const created = await createUser(body)
  setResponseStatus(event, 201)
  return created
})
```

```ts
// server/api/users/[id].get.ts  →  GET /api/users/123（動的パラメータ）
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id')
  return await findUser(id)
})
```

```ts
// server/routes/sitemap.xml.ts  →  /sitemap.xml（/api を付けない素のルート）
export default defineEventHandler((event) => {
  setHeader(event, 'content-type', 'application/xml')
  return '<?xml version="1.0"?>...'
})
```

```ts
// server/middleware/auth.ts  →  全リクエストで先に走る共通処理
export default defineEventHandler((event) => {
  // ここで認証チェックやログを仕込む（return しなければ後続へ続く）
  event.context.requestId = crypto.randomUUID()
})
```

## 実務での使い方・定番パターン
- **`server/api/`** … `/api/*` で叩くJSON API置き場。ファイル名でメソッドを分けるのが基本（`.get` / `.post` / `.put` / `.delete` / `.patch`）。
- **`server/routes/`** … `/api` プレフィックスなしのルート。`robots.txt` や `sitemap.xml`、Webhook受け口などに使う。
- **`server/middleware/`** … 全リクエスト共通の前処理（認証・CORS・ロギング）。**ファイルを置くだけで全ルートに自動適用**される。
- **戻り値＝レスポンス**。オブジェクトを `return` すれば自動でJSON。エラーは `throw createError({ statusCode, statusMessage })` で返す。
- ヘルパーは `h3` 由来：`getQuery` / `readBody` / `getRouterParam` / `getHeader` / `setHeader` / `setResponseStatus` / `getCookie` / `setCookie`。
- クライアントからは **`useFetch('/api/users')`** で取得し、SSR時はサーバ内で完結。→ [data_fetching.md](./data_fetching.md)
- ここはサーバ専用なので、**シークレットへのアクセスはOK**。`useRuntimeConfig(event)` の非public値を読める。→ [config.md](./config.md)

## ハマりどころ / アンチパターン
- **HTTPメソッドはファイル名のサフィックスで決まる**：`users.ts` は全メソッド受けだが、`users.get.ts` はGET専用。POSTしたいのにGET用ファイルしかない、で405になりがち。意図したメソッドサフィックスを必ず付ける。
- **`defineEventHandler` で包むのが必須**。素の関数を `export default` しても動かない。`return defineEventHandler(...)` でなく `export default defineEventHandler(...)`。
- **`readBody` は1回だけ**読める前提で扱う（ストリーム消費）。同じ event で複数回読もうとしない。
- **ここに書いたコードはブラウザに出ない**＝クライアントの `composables/` や `components/` からは直接 import できない。共有したいロジックは `server/utils/`（サーバ自動インポート）に置く。クライアントと混同しない。
- **`server/api/` 内で `useState` や Vue の composable を呼ばない**。これらは Vue/Nuxt のクライアント/SSRレンダリング文脈用。サーバルートは h3 の `event` で完結させる。
- **重い同期処理でイベントループを塞がない**。外部API/DBは `await` で非同期に。レート制限や入力バリデーションも忘れず（境界での検証）。

## 関連
[data_fetching.md](./data_fetching.md) / [config.md](./config.md) / [deployment.md](./deployment.md)
