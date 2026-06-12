# 設定（nuxt.config / runtimeConfig / modules）（Nuxt 3）

## ひとことで言うと
プロジェクト全体の挙動を決める**唯一の設定ファイル `nuxt.config.ts`** と、そこにぶら下がる**実行時設定 `runtimeConfig`**・**モジュール `modules`** のこと。ビルド設定・モジュール有効化・環境変数の受け渡しを一箇所に集約する。

## 役割・なぜ必要か
- Nuxt は規約（ディレクトリ構成）で動くが、それを**上書き・拡張する入口**が `nuxt.config.ts`。SSR の ON/OFF、エイリアス、CSS、モジュールなどをここで宣言する。
- **秘密情報（APIキー等）をコードに直書きしない**ために `runtimeConfig` がある。ここに入れた値だけが「サーバ専用」と「クライアント公開」に分離され、安全に環境変数で差し替えられる。
- `modules` は Nuxt の機能を増やす標準の手段（例: `@nuxt/image`・`@pinia/nuxt`）。自前で webpack/vite をいじらず、モジュールを足すだけで機能が増える。

## 基本の書き方（コード）
```ts
// nuxt.config.ts
export default defineNuxtConfig({
  // 開発支援ツール（Nuxt DevTools）
  devtools: { enabled: true },

  // 機能拡張（標準の足し方）
  modules: ['@pinia/nuxt', '@nuxt/image'],

  // 実行時設定（環境変数で上書き可能）
  runtimeConfig: {
    // ↓ サーバ専用（クライアントへは絶対に出ない）
    apiSecret: '',            // 既定は空。実値は .env から注入
    db: { url: '' },

    // ↓ public はクライアントにも露出する（公開前提のものだけ）
    public: {
      apiBase: '/api',        // フロントから叩く公開URL
      siteUrl: 'http://localhost:3000',
    },
  },

  // 全ページ共通CSS / エイリアスなど
  css: ['~/assets/css/main.css'],
})
```

```bash
# .env … NUXT_ 接頭辞で runtimeConfig を「キー名に対応させて」上書きする
# 規則: NUXT_<KEY>  /  ネストは _ で連結  /  public は NUXT_PUBLIC_<KEY>
NUXT_API_SECRET=sk_live_xxxxxxxx          # → runtimeConfig.apiSecret
NUXT_DB_URL=postgres://user:pass@host/db  # → runtimeConfig.db.url
NUXT_PUBLIC_API_BASE=https://api.example.com   # → runtimeConfig.public.apiBase
NUXT_PUBLIC_SITE_URL=https://example.com
```

```ts
// 取り出し方（読むのは useRuntimeConfig）
// サーバ側（server/api 等）: public も非public も読める
export default defineEventHandler(() => {
  const config = useRuntimeConfig()
  const secret = config.apiSecret          // サーバでのみ参照可
  return { ok: !!secret }
})

// クライアント側（components/composables）: public のみ読める
const config = useRuntimeConfig()
const base = config.public.apiBase         // OK。config.apiSecret は undefined
```

## 実務での使い方・定番パターン
- **秘密は必ず `runtimeConfig`（非public）に置く**。`public` は「漏れてもよい公開値」専用（公開API URL・サイトURL・GA測定IDなど）。
- 既定値はコード（`nuxt.config.ts`）に**空文字や開発用の値**だけ書き、**実値は `.env`（＝環境変数）から注入**する。これでリポジトリに秘密を残さない。
- `.env` の変数名は **`NUXT_` 接頭辞 + キーの大文字スネーク** に揃える。ネストは `_` 連結、public は `NUXT_PUBLIC_`。命名が一致しないと**黙って反映されない**ので要注意。
- 値の参照は常に **`useRuntimeConfig()`** 経由。`process.env` を直接読むのは避ける（クライアントに出ない・型が付かない・SSR で挙動が割れる）。
- モジュールの設定は、そのモジュール専用のトップレベルキー（例: `image: {}`・`pinia: {}`）か `modules` 配列に書く。順序が意味を持つモジュールもあるので公式の指定に従う。
- 環境ごとの差し替えは `.env`／`.env.production` とデプロイ先の環境変数で行う。コードの分岐より環境変数が基本。
- **モジュールは「足すだけ」が原則**。`@pinia/nuxt`（状態）・`@nuxt/image`（画像最適化）・`@vueuse/nuxt`（ユーティリティ）など定番を `modules` に並べる。自前で vite/webpack を触る前に、まず対応モジュールが無いか探す。
- `app.head`・`css`・`alias`（パス別名）・`vite`（ビルダ設定）など、設定はトップレベルキーに集約。1ファイルで全体像が見えるのが Nuxt の利点なので、無闇に分散させない。

## ハマりどころ / アンチパターン
- **秘密を `runtimeConfig.public` に入れる**＝最大の事故。public はクライアントバンドルに埋め込まれ、ブラウザの DevTools から丸見えになる。秘密は必ず非public側へ。
- **`.env` の接頭辞ミス**：`API_SECRET=...`（接頭辞なし）では `runtimeConfig` を上書きできない。必ず `NUXT_API_SECRET`。ネスト名の `_` 連結も忘れがち。
- **ビルド時 vs ランタイムの混同**：`runtimeConfig` は**起動時**に環境変数で解決される（同じビルド成果物を環境ごとに使い回せる）。一方 `process.env` の直接参照や `import.meta.env` はビルド時に固定されがちで、本番で値が変わらない事故になる。差し替えたい値は `runtimeConfig`。
- **サーバ専用値をクライアントで読もうとする**：`config.apiSecret` はクライアントでは `undefined`。サーバ側（`server/api`・middleware のサーバ実行など）でだけ参照する。→ [server_routes.md](./server_routes.md)
- **`.env` をコミットする**：`.gitignore` 必須。サンプルは `.env.example` にキー名だけ残す。
- **デプロイ先で環境変数を設定し忘れる**：ローカルは `.env` で動くが、本番は各プラットフォームの環境変数設定が必要。→ [deployment.md](./deployment.md)

## 関連
[server_routes.md](./server_routes.md) / [deployment.md](./deployment.md) / [rendering.md](./rendering.md)
