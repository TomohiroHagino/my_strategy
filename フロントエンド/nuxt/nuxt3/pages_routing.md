# ページ / ファイルベースルーティング（Nuxt 3）

## ひとことで言うと
**`pages/` ディレクトリに置いた `.vue` ファイルの「パスと名前」が、そのまま URL になる**仕組み。ルータ設定を手書きせず、ファイル配置でルーティングが決まる。

## 役割・なぜ必要か
- Vue Router を自前で `routes: [...]` と書く代わりに、**ファイル＝ルート**にすることで設定とファイルがズレない。
- 「どの URL がどのコンポーネントか」を **フォルダを見れば把握できる**。新規ページの追加＝ファイル追加だけ。
- 動的パラメータ（`/users/123`）やネスト、catch-all も **ファイル名の規約**で表現できる。ルーティングの知識をファイル命名へ集約できるのが利点。

## 基本の書き方（コード）
```bash
pages/
├─ index.vue            # "/"
├─ about.vue            # "/about"
├─ users/
│   ├─ index.vue        # "/users"
│   └─ [id].vue         # "/users/:id"  動的（/users/123）
├─ blog/
│   └─ [...slug].vue    # "/blog/*"     catch-all（/blog/a/b/c）
└─ parent.vue + parent/ # ネスト（parent.vue に <NuxtPage/>）
```

```vue
<!-- pages/users/[id].vue：動的パラメータを取る -->
<script setup lang="ts">
const route = useRoute()
const id = route.params.id          // "/users/123" → "123"
</script>

<template>
  <div>ユーザーID: {{ id }}</div>
</template>
```

```vue
<!-- pages/index.vue：リンクとページメタ -->
<script setup lang="ts">
definePageMeta({
  layout: 'default',                // 使うレイアウト
  middleware: 'auth',               // ルートミドルウェア
})
function go() {
  navigateTo('/users/123')          // JS から遷移
}
</script>

<template>
  <nav>
    <!-- 内部遷移は必ず NuxtLink（フルリロードしない） -->
    <NuxtLink to="/about">About</NuxtLink>
    <NuxtLink :to="`/users/${1}`">User 1</NuxtLink>
    <button @click="go">プログラムで遷移</button>
  </nav>
</template>
```

## 実務での使い方・定番パターン
- **`<NuxtLink>` で内部遷移**。クライアント側でルートだけ差し替わり（SPA 的に高速）。外部 URL や `mailto:` は普通の `<a>` で良い。
- **`useRoute()` でパラメータ・クエリを読む**：`route.params.id`、`route.query.page`。書き換えは `navigateTo({ query: { page: 2 } })`。
- **`definePageMeta`** で「このページ専用」のレイアウト・ミドルウェア・`name`・`alias` を宣言。SFC の `<script setup>` 内に書く。
- **catch-all `[...slug]`** はCMS連携や 404 的なフォールバックに。`route.params.slug` は配列で来る（`['a','b','c']`）。
- **ネスト**：`pages/parent.vue` に `<NuxtPage />` を置くと、`pages/parent/child.vue` がその中に描画される（タブUIなど）。
- **プログラム遷移は `navigateTo`**。`return navigateTo('/login')` のように `return` するとミドルウェア内でリダイレクトになる。→ [layouts_middleware.md](./layouts_middleware.md)

## ハマりどころ / アンチパターン
- **`pages/` が無いとルーティングそのものが無効**。その時アプリは `app.vue` 1画面だけ。`/about` を作りたいのに `pages/` を置いていない、が最頻出。→ [getting_started.md](./getting_started.md)
- **`app.vue` に `<NuxtPage />` が無い**と、`pages/` を作っても表示されない。`pages/` を使うなら `app.vue` 側に `<NuxtPage />`（または `<NuxtLayout>`）が必須。
- **内部遷移に `<a href>` を使う**とフルリロードが起きる（状態が飛ぶ・遅い）。内部は必ず `<NuxtLink>`。
- **`[id]` の角括弧をファイル名から落とす**（`id.vue` にしてしまう）と動的にならず、`/users/id` という固定URLになる。
- **`useRoute()` を `<script setup>` の外**（普通の関数の中など）で呼ぶと動かないことがある。セットアップ直下で呼ぶ。
- **catch-all のパラメータを文字列扱い**してエラー。`slug` は **配列**。`.join('/')` で結合する。
- `navigateTo` の戻り値を `await`/`return` せず放置すると、リダイレクトが効かない場面がある。

## 関連
[layouts_middleware.md](./layouts_middleware.md) / [getting_started.md](./getting_started.md) / [components_autoimport.md](./components_autoimport.md)
