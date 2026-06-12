# SEO / メタ（useHead / useSeoMeta）（Nuxt 3）

## ひとことで言うと
ページの `<head>`（title・meta・link 等）を**コンポーネントから宣言的に書き換える仕組み**。汎用の **`useHead`** と、OG/Twitter などを型付きで安全に書ける **`useSeoMeta`** がある。Nuxt は SSR でこれらをサーバ側 HTML に焼き込むので**SEO に効く**。

## 役割・なぜ必要か
- 検索エンジンや SNS クローラは**最初に返る HTML の `<head>`** を読む。SPA でクライアント描画後に title を書き換えても、クローラが見るのは初期 HTML なので不利になる。Nuxt は **SSR で head をサーバレンダ**するため、適切に書けばそのまま評価される。
- `useHead` は title・meta・link・script など**あらゆる head 要素**を扱える低レベルAPI。
- `useSeoMeta` は OGP（`og:title` 等）・Twitter Card・description などを**プロパティ名で型安全に**書くための専用API。タイポや属性名ミス（`property` vs `name`）を防げる。
- ページ単位のメタ情報（layout 指定・middleware など）は **`definePageMeta`** で宣言する。

## 基本の書き方（コード）
```vue
<script setup lang="ts">
// 1) 汎用: useHead（title / meta / link を直接組み立てる）
useHead({
  title: '記事一覧',
  titleTemplate: (t) => (t ? `${t} | MySite` : 'MySite'),
  meta: [{ name: 'description', content: '最新記事の一覧ページです' }],
  link: [{ rel: 'canonical', href: 'https://example.com/articles' }],
  htmlAttrs: { lang: 'ja' },
})

// 2) 型付き: useSeoMeta（OG / Twitter を安全に。属性ミスが起きない）
useSeoMeta({
  title: '記事一覧',
  description: '最新記事の一覧ページです',
  ogTitle: '記事一覧 | MySite',
  ogDescription: '最新記事の一覧ページです',
  ogImage: 'https://example.com/og/articles.png',
  ogType: 'website',
  twitterCard: 'summary_large_image',
})
</script>
```

```vue
<script setup lang="ts">
// 動的メタ: 値が後から決まる場合は「関数（getter）」で渡す
// → リアクティブに追従し、SSR でも最新値がレンダされる
const route = useRoute()
const { data: post } = await useFetch(`/api/posts/${route.params.id}`)

useSeoMeta({
  title: () => post.value?.title ?? '読み込み中',
  description: () => post.value?.excerpt ?? '',
  ogImage: () => post.value?.coverUrl,
})
</script>
```

```vue
<script setup lang="ts">
// definePageMeta: ページ単位のメタ（layout / middleware / 任意キー）
definePageMeta({
  layout: 'blog',
  middleware: ['auth'],
})
// 注意: definePageMeta は head を書かない。head は useHead/useSeoMeta で。
</script>
```

```ts
// nuxt.config.ts: 全ページ共通の既定 head はここに集約できる
export default defineNuxtConfig({
  app: {
    head: {
      htmlAttrs: { lang: 'ja' },
      titleTemplate: '%s | MySite',
      meta: [{ name: 'viewport', content: 'width=device-width, initial-scale=1' }],
    },
  },
})
```

## 実務での使い方・定番パターン
- **共通の既定は `app.head`（nuxt.config）**、**ページ固有は各ページの `useSeoMeta`/`useHead`** に分ける。重複定義は後勝ちでマージされる。
- OG/Twitter は基本 **`useSeoMeta`** を使う。`property="og:..."` を手書きする `useHead` よりタイポしにくく型補完が効く。
- `titleTemplate` でサイト名サフィックスを一元化（`'%s | MySite'`）。各ページは `title` だけ書けばよい。
- `canonical`（正規URL）と `og:image` は SEO/SNS で効果が大きい。記事ページでは**動的に**設定する。
- 動的値は**必ず関数（getter）で渡す**。`post.value.title` を即値で渡すと、フェッチ前の値で固定されてしまう。
- 構造化データ（JSON-LD）が必要なら `useHead({ script: [{ type: 'application/ld+json', innerHTML: ... }] })` で追加できる。

## ハマりどころ / アンチパターン
- **SSR 前提を忘れる**：SEO が効くのは**サーバレンダされた初期HTML**にメタが入るから。該当ルートを `ssr: false`（SPA化）にすると、クローラはメタを読めず効果が激減する。SEO 重視ページは SSR/SSG を維持する。→ [rendering.md](./rendering.md)
- **動的メタを即値で渡す**：`title: post.value.title` はフェッチ完了前だと空。`title: () => post.value?.title` と**関数**にする。
- **`definePageMeta` で head を書こうとする**：これは layout/middleware 等の宣言用で、title/meta は書けない。head は `useHead`/`useSeoMeta` を使う。
- **`useSeoMeta` と `useHead` の二重定義で衝突**：同じ項目を両方に書くと後勝ちで混乱する。OG 系は `useSeoMeta` に寄せ、その他 link/script を `useHead` で補う、と役割を分ける。
- **`og:image` を相対パスにする**：SNS は絶対URLを要求する。`https://...` の完全URLで指定する（`runtimeConfig.public.siteUrl` と組み合わせると堅い）。
- **クライアントでしか取れない値に依存**：`window` 依存の値で head を作るとSSR時に欠落する。サーバでも解決できる値（`useFetch` 等）から組む。

## 関連
[rendering.md](./rendering.md) / [data_fetching.md](./data_fetching.md) / [config.md](./config.md)
