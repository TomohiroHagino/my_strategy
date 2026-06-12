# デプロイ（Nitro presets）（Nuxt 3）

## ひとことで言うと
Nuxt のサーバエンジン **Nitro** が、**preset（出力ターゲット）**を切り替えるだけで Node サーバ・各種クラウド・静的サイトなど**どこへでも同じコードをデプロイできる**仕組み。`nuxi build`（SSR）と `nuxi generate`（SSG）の2系統がある。

## 役割・なぜ必要か
- 通常、デプロイ先（Vercel / Netlify / Cloudflare / 自前 Node）ごとに必要な成果物の形は違う。Nitro は**同じソースから preset に応じた成果物を自動生成**するので、移行や乗り換えが容易。
- **SSR**（リクエストごとにサーバで HTML 生成）と **SSG**（ビルド時に全ページを静的HTML化）という2つの配信戦略を、ほぼ同じコードのまま切り替えられる。
- `runtimeConfig`（→ [config.md](./config.md)）と組み合わせ、**ビルド成果物は1つ・環境変数だけ差し替え**という運用ができる（SSR時）。

## 基本の書き方（コード）
```bash
# SSR ビルド（Nodeサーバ等で動かす動的サイト）
npx nuxi build
# → .output/ が生成される。Node なら下記で起動:
node .output/server/index.mjs        # 既定で http://localhost:3000

# SSG ビルド（全ページを静的HTMLに事前生成。サーバ不要で配信）
npx nuxi generate
# → .output/public/ に静的ファイル一式。これをそのまま配信する
```

```ts
// nuxt.config.ts: preset を明示（多くのプラットフォームは自動検出だが明示が安全）
export default defineNuxtConfig({
  nitro: {
    // 代表的な preset:
    //  'node-server'  … 汎用Nodeサーバ（self-host / Docker）
    //  'vercel'       … Vercel
    //  'netlify'      … Netlify
    //  'cloudflare-pages' / 'cloudflare-module' … Cloudflare
    //  'static'       … 静的出力（nuxi generate 相当）
    preset: 'node-server',
  },
})
```

```ts
// nuxt.config.ts: ルート単位でレンダ方式を混在（Hybrid Rendering）
export default defineNuxtConfig({
  routeRules: {
    '/':            { prerender: true },              // 静的に事前生成
    '/blog/**':     { isr: 3600 },                    // ISR: 1時間ごと再生成
    '/admin/**':    { ssr: true },                    // 都度SSR
    '/dashboard/**':{ ssr: false },                   // SPA（クライアント描画）
    '/old-path':    { redirect: '/new-path' },        // リダイレクト
  },
})
```

```bash
# Dockerでの self-host 例（node-server preset）
# 1) ビルド
npx nuxi build
# 2) 本番に必要なのは .output/ だけ。環境変数を渡して起動する
NUXT_API_SECRET=xxx NUXT_PUBLIC_API_BASE=https://api.example.com \
  node .output/server/index.mjs
```

## 実務での使い方・定番パターン
- **配信先が決まっているなら preset を明示**（`nitro.preset`）。CI で確実に同じ成果物が出る。Vercel/Netlify/Cloudflare は多くの場合自動検出されるが、明示しておくと事故が減る。
- **動的データ・認証・サーバAPIがある → SSR（`nuxi build`）**。`server/api`（→ [server_routes.md](./server_routes.md)）を使うならサーバが要るので SSR 系 preset。
- **完全に静的（ブログ・LP・ドキュメント）→ SSG（`nuxi generate`）**。CDN/静的ホスティングに置くだけで動き、運用が軽い。
- **混在させたい → `routeRules`**。トップは prerender、管理画面は SSR、一覧は ISR…とルートごとに最適化できる（Hybrid Rendering）。→ [rendering.md](./rendering.md)
- **環境変数はプラットフォーム側で設定**。`runtimeConfig` の値は `NUXT_` 接頭辞の環境変数で起動時に注入する。SSR なら**同じビルドを本番/ステージングで使い回せる**。→ [config.md](./config.md)
- **secret はビルドログ・クライアントに出さない**。`runtimeConfig`（非public）に置き、各環境の secret manager / 環境変数で渡す。
- **プレビュー確認**は `nuxi preview`（`nuxi build` 後にローカルで本番相当を起動）で行うと、デプロイ前に SSR 成果物の動作を確かめられる。

## ハマりどころ / アンチパターン
- **preset の選択ミス**：Cloudflare Workers のような環境では Node API の一部が使えない。`server/api` で Node 専用機能（特定のネイティブモジュール等）を使うと preset によっては動かない。デプロイ先の制約を先に確認する。
- **SSG なのに動的データを期待**：`nuxi generate` はビルド時点のデータで固定される。ユーザーごと・リクエストごとに変わる内容は静的化できない。動的が要るなら SSR か ISR/`routeRules` を使う。
- **SSG で動的ルートが生成されない**：`/posts/[id]` のような動的ページは、prerender 対象として**クロール元のリンクか `nitro.prerender.routes` で明示**しないと生成されない。
- **環境変数の設定忘れ**：ローカルは `.env` で動くが、本番プラットフォームで `NUXT_...` を設定し忘れると `runtimeConfig` が空のままになり、API認証などが落ちる。→ [config.md](./config.md)
- **`runtimeConfig` をビルド時固定と勘違い**：SSR では起動時に環境変数で解決されるが、**SSG（静的出力）ではビルド時に焼き込まれる**。SSG で値を変えたいなら再ビルドが必要。
- **`.output/` を編集してデプロイ**：生成物は触らない。設定は `nuxt.config.ts` で行い、再ビルドする。

## 関連
[config.md](./config.md) / [rendering.md](./rendering.md) / [server_routes.md](./server_routes.md)
