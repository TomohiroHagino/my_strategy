# 描画（SSR / SSG / SPA・ハイドレーション）（Nuxt 3）

## ひとことで言うと
**描画モード** = HTMLを「いつ・どこで」作るかの戦略。Nuxt 3は既定で**ユニバーサル（SSR）**＝サーバでHTMLを生成し、ブラウザでVueを再アタッチ（ハイドレーション）して動かす。

## 役割・なぜ必要か
- **SSR（既定）**：サーバで完成HTMLを返すので初回表示が速く、SEO（クローラがHTMLを読める）に強い。その後 `app.vue` 以下を**ハイドレーション**してインタラクティブにする。
- **SSG（静的生成）**：ビルド時に各ページをHTMLへ焼く（`nuxi generate` / `prerender`）。サーバ不要でCDN配信でき、内容が頻繁に変わらないサイト（ブログ・LP）に最適。
- **SPA（`ssr: false`）**：HTMLは空シェルだけ返し、描画は全てブラウザで行う。SEO不要な管理画面・社内ツール向き。
- 「**最初のHTMLを誰が作るか**（サーバ/ビルド/ブラウザ）」が違うだけで、これが初速・SEO・インフラ要件を決める。

## 基本の書き方（コード）
```ts
// nuxt.config.ts : 描画モードの切替
export default defineNuxtConfig({
  ssr: true, // 既定。false にすると全体 SPA（ブラウザ描画のみ）

  // ルート単位で混在も可能（ハイブリッドレンダリング）
  routes: {
    '/admin/**': { ssr: false },          // 管理画面だけ SPA
    '/blog/**':  { prerender: true },     // ブログはビルド時に静的化（SSG）
    '/news/**':  { swr: 600 },            // 600秒キャッシュ（ISR風）
  } as any,
})
```

```bash
# SSG : 全ページを静的HTMLへ書き出す（dist/ をCDNに置くだけ）
npx nuxi generate
# SSR サーバとして動かす（Nitro サーバ出力）
npx nuxi build && node .output/server/index.mjs
```

```vue
<script setup lang="ts">
// ブラウザAPIはクライアントでのみ触る（サーバには window が無い）
const width = ref(0)
onMounted(() => {
  width.value = window.innerWidth // onMounted はクライアント専用なので安全
})
// 条件分岐したいとき
if (import.meta.client) {
  // ここはブラウザでのみ実行される
}
</script>

<template>
  <p>幅: {{ width }}</p>

  <!-- サーバで描画させたくない／ブラウザ依存の塊は ClientOnly で包む -->
  <ClientOnly>
    <InteractiveChart />
    <template #fallback><p>読み込み中…</p></template> <!-- SSR時の代替表示 -->
  </ClientOnly>
</template>
```

## 実務での使い方・定番パターン
- **既定はSSRのまま**：多くのサイトはSSRで初速＋SEOを取りつつ、必要箇所だけ調整するのが基本。
- **ハイブリッド**：公開ページはSSR/SSG、ログイン後の管理画面は `ssr: false` のSPA、と `routes` で混在させる。
- **SSGの境界**：動的データが少なく更新頻度が低いなら `prerender: true`。ビルド時に存在しないURL（DB由来の動的ルート）は `nitro.prerender.routes` で列挙するか `crawlLinks` で辿らせる。
- **`swr` / ISR**：完全静的にしたくないがリクエスト毎の生成も重い、なら `swr`（秒数キャッシュ）で「速さ」と「鮮度」を両立。
- **ブラウザ依存UIは `<ClientOnly>`**：地図・チャート・`window` を読むウィジェットはこれで包み、SSRでは `#fallback` を出す。
- **データ取得と一体で考える**：SSRに焼くデータは `useFetch`、ブラウザ専用データは `server: false`。→ [data_fetching.md](./data_fetching.md)

## ハマりどころ / アンチパターン
- **ハイドレーションミスマッチ**：サーバが作ったDOMとクライアントの初回描画が食い違うと、Vueが「Hydration node mismatch」警告を出し、その部分が再描画されたり壊れたりする。原因の典型は **`new Date()`/`Math.random()`/`Date.now()`** をテンプレートで直接使う（サーバ時刻とクライアント時刻が違う）こと。固定値や `useState` でサーバ値を引き継ぐ。
- **`window`/`document` をサーバで触る**：`<script setup>` のトップレベルや SSR で走る箇所で `window.localStorage` 等を読むと**サーバ側で `window is not defined` で落ちる**。`onMounted` か `import.meta.client` ガード、または `<ClientOnly>` で囲う。
- **`localStorage` で状態を初期化**：サーバには無いのでミスマッチの温床。SSR安全な共有状態は `useState`、ブラウザ跨ぎの永続は `useCookie` を使う。→ [state.md](./state.md)
- **`v-if="import.meta.client"` で出し分け**：サーバでは描かずクライアントで描く＝差分が出てミスマッチ警告になりやすい。意図的な出し分けは `<ClientOnly>` を使う（Nuxtがミスマッチ扱いしない）。
- **SSGなのに動的データ前提**：`nuxi generate` はビルド時の値で固定されるため、ユーザー毎に変わる内容は静的化に向かない。そこはSSRかクライアント取得へ。
- **巨大コンポーネントを全部SSR**：重いブラウザ専用ライブラリまでサーバで読もうとして失敗。動的 import ＋ `<ClientOnly>` で分離する。

## 関連
[data_fetching.md](./data_fetching.md) / [state.md](./state.md) / [deployment.md](./deployment.md)
