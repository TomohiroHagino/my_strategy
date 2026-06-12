# 始め方（Getting Started）（Hono v4）

## ひとことで言うと
Hono の最小アプリ（`new Hono()` → ルート定義 → 起動）を作る手順。**ランタイムごとに「起動の仕方」だけが違う**のがポイント。

## 役割・なぜ必要か
- Hono は「**マルチランタイムの薄いルータ + Context**」なので、アプリ本体（ルート定義）はどこでも同じ書き方になる。
- 違うのは**エントリ（どう起動するか）**だけ：Workers / Bun / Deno は `export default app`、Node は `@hono/node-server` の `serve()` が要る。
- `npm create hono@latest` のテンプレ選択で、その差分（エントリ・設定・依存）を自動で用意してくれる。最初はテンプレに乗るのが正解。

## 基本の書き方（コード）
```bash
# 対話式でテンプレを選ぶ（cloudflare-workers / nodejs / bun / deno など）
npm create hono@latest my-app
cd my-app
npm install
npm run dev        # テンプレが用意した dev サーバが立つ
```

```ts
// src/index.ts … アプリ本体は全ランタイム共通
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.json({ message: 'Hello Hono!' }))
app.get('/health', (c) => c.text('ok'))

export default app   // ← Workers / Bun / Deno はこれで起動
```

```ts
// Node の場合だけ「起動コード」を足す（@hono/node-server が必要）
import { serve } from '@hono/node-server'
import { Hono } from 'hono'

const app = new Hono()
app.get('/', (c) => c.json({ message: 'Hello Hono!' }))

serve({ fetch: app.fetch, port: 3000 }, (info) => {
  console.log(`Listening on http://localhost:${info.port}`)
})
```

## 実務での使い方・定番パターン
- **テンプレで雛形を作る**：ランタイムを決めたら `npm create hono@latest` 一発。`cloudflare-workers` なら `wrangler.toml`、`nodejs` なら `serve()` 入りの雛形が出る。
- **アプリ本体（ルート）とエントリを分ける**：`src/app.ts` で `export const app = new Hono()` を作り、`src/index.ts`（or `worker.ts`）で起動だけ行う。→ ランタイム差し替え・テストが楽。
- **`package.json` の `type: "module"`**：Hono は ESM 前提。Node でも `"type": "module"` にしておく。
- **TypeScript 設定**：`tsconfig.json` の `"jsx": "react-jsx"`・`"jsxImportSource": "hono/jsx"`（JSX を使うなら）、`"moduleResolution": "Bundler"` あたりが定番。テンプレが入れてくれる。
- **dev は各ランタイムのツールで**：Workers は `wrangler dev`、Bun は `bun run`、Node は `tsx watch`。`npm run dev` の中身がランタイムで違うだけ。

## ハマりどころ / アンチパターン
- **`export default app` を Node でやって動かない**：Node は HTTP サーバを自前で起動しないと listen しない。`@hono/node-server` の `serve({ fetch: app.fetch })` が必須。逆に **Workers/Bun で `serve()` を書くのは不要**（むしろ誤り）。
- **ランタイムを後から変える前提でテンプレ依存を混ぜる**：Workers テンプレには `wrangler` 前提のコードが、Node テンプレには `serve()` が入る。**最初にランタイムを決める**のが一番事故が少ない。→ [runtimes.md](./runtimes.md)
- **`serve` を `hono` 本体から import しようとする**：`serve` は `hono` ではなく `@hono/node-server`。`hono` 本体にサーバ起動 API は無い。
- **CommonJS のまま動かす**：`require` 環境だと ESM 専用の Hono が読めずエラー。`"type": "module"` を入れる。
- **TS の `jsx` 設定を React のまま**にして `hono/jsx` が壊れる：JSX を使うなら `jsxImportSource` を `hono/jsx` に。
- **`c.json()` の `return` 忘れ**：ハンドラは Response を**返す**必要がある（`return c.json(...)`）。返さないと空レスポンスになる。→ [context.md](./context.md)

## フォルダ構成（始動直後）
> **Hono に公式の決まった構成は無い**（最小は `src/index.ts` 1枚でも動く）。
> 下は規模が出たときの定番の層構成で、**`src/` の中身はほぼ全部自分で作る**。

```
myapp/
├── src/
│   ├── index.ts          # エントリ（new Hono() → ルート登録 → export default app）
│   ├── routes/           # ルート定義（パス↔ハンドラの結びつけ）   # 自分で作る
│   │   └── users.ts      #   例: app.route('/users', usersRoute)
│   ├── handlers/         # ハンドラ（= controller。c でreq受取・レス返却） # 自分で作る
│   │   └── user.ts       #   例: (c) => c.json(...)
│   ├── services/         # 業務ロジック                          # 自分で作る
│   │   └── user.ts
│   ├── repositories/     # DBアクセス（or db/）                   # 自分で作る
│   │   └── user.ts
│   ├── middleware/       # 認証・ログ等の自作ミドルウェア          # 自分で作る
│   │   └── auth.ts
│   ├── schemas/          # 入出力バリデーション（zod等）           # 自分で作る
│   │   └── user.ts
│   └── types.ts          # 共有の型定義                          # 自分で作る
├── package.json          # 依存・scripts（dev 等）・"type":"module"
├── tsconfig.json         # TS設定（moduleResolution: Bundler 等）
├── wrangler.toml         # Cloudflare Workers の設定【Workers時】（Node なら無し）
├── .dev.vars             # Workers のローカル秘密値【Workers時】   # 自分で作る
├── .gitignore            # node_modules / .dev.vars 等の除外       # 自分で作る
└── node_modules/         # 依存の実体（npm install で生成）
```
- **層の発想は Spring と同じ**：`routes`(結びつけ) → **`handlers`(= controller・`c` で req/res)** → `services`(業務) → `repositories`(DB)。
- バリデーションは **zod + `@hono/zod-validator`**（`schemas/`）。検証済み値は `c.req.valid('json')` で型付きで取れる。
- 実行環境（Cloudflare Workers / Node / Bun / Deno）をテンプレ選択時に決める。**実行環境で設定ファイルが変わる**（Workers=`wrangler.toml`+`.dev.vars`、Node=`@hono/node-server`、Bun/Deno=各設定）。TypeScript前提。

## 関連
[routing.md](./routing.md) / [runtimes.md](./runtimes.md) / [context.md](./context.md)
