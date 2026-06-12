# ルーティング（Vue Router）（Vue 3）

## ひとことで言うと
URL と画面（コンポーネント）を対応づけ、**ページ遷移をリロードなしで切り替える** Vue 公式のルーター。`createRouter` でルート定義を作り、`<RouterView>` に該当コンポーネントを描画、`<RouterLink>` でリンクを張る。

## 役割・なぜ必要か
- SPA（単一ページアプリ）は HTML を1枚しか読み込まない。その中で「`/users/1` を開いたらユーザー詳細を表示」のように、**URL に応じて表示するコンポーネントを差し替える**仕組みが必要になる。それを担うのが Vue Router。
- URL を状態として使えるので、**ブックマーク・履歴（戻る/進む）・共有**が成立する。検索条件やタブを URL に持たせれば再現可能になる。
- 遷移前後に割り込む**ナビゲーションガード**で、認証チェックや確認ダイアログなど「**遷移してよいか**」の制御を一元化できる。

## 基本の書き方（コード）
```ts
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import Home from '@/pages/Home.vue'

const router = createRouter({
  // createWebHistory: 普通のURL（/users/1）。GitHub Pages等は createWebHashHistory も
  history: createWebHistory(),
  routes: [
    { path: '/', name: 'home', component: Home },
    // 遅延読み込み（コード分割）。大きい画面はこの形が定番
    { path: '/about', component: () => import('@/pages/About.vue') },
    // 動的ルート: :id がパラメータになる
    { path: '/users/:id', name: 'user', component: () => import('@/pages/User.vue') },
  ],
})

// グローバルなナビゲーションガード（認証など）
router.beforeEach((to) => {
  const isAuthed = Boolean(localStorage.getItem('token'))
  if (to.meta.requiresAuth && !isAuthed) return { name: 'home' }  // リダイレクト
  // 何も返さない/true なら遷移を許可
})

export default router
```
```ts
// main.ts
import router from './router'
app.use(router).mount('#app')
```
```vue
<!-- App.vue: リンクと表示枠 -->
<template>
  <nav>
    <RouterLink to="/">Home</RouterLink>
    <RouterLink :to="{ name: 'user', params: { id: 1 } }">User 1</RouterLink>
  </nav>
  <RouterView />   <!-- ここに該当ページが描画される -->
</template>
```
```vue
<!-- pages/User.vue: パラメータ取得とプログラム遷移 -->
<script setup lang="ts">
import { useRoute, useRouter } from 'vue-router'
const route = useRoute()      // 現在のルート情報（読み取り専用）
const router = useRouter()    // 遷移の操作

const id = route.params.id    // '/users/1' → '1'（文字列）
function goHome() { router.push('/') }                 // パスで
function goUser(n: number) { router.push({ name: 'user', params: { id: n } }) }  // 名前で
</script>
```

## 実務での使い方・定番パターン
- **名前付きルート（`name`）で遷移**：パス文字列をベタ書きするより、`{ name, params }` で指定するとパス変更に強い。
- **動的ルートのパラメータ**は `useRoute().params` で取得。同じコンポーネントのまま `id` だけ変わる遷移（`/users/1`→`/users/2`）では `watch(() => route.params.id, ...)` で再取得する。→ [computed_watch.md](./computed_watch.md)
- **ネストルート・レイアウト**：子ルートを定義し、親テンプレートに `<RouterView>` を置くと入れ子表示になる。
- **ガードの使い分け**：全体は `router.beforeEach`、特定ページは route の `beforeEnter`、コンポーネント内は `onBeforeRouteLeave`（離脱確認など）。認証状態は store と組み合わせることが多い。→ [pinia.md](./pinia.md)
- **active クラス**：`<RouterLink>` は現在地に応じて `router-link-active` などが自動付与され、ナビの強調に使える。

## ハマりどころ / アンチパターン
- **`<a href>` で内部遷移する**：素の `<a>` は**ページ全体をリロード**し SPA の状態が消える。内部遷移は必ず `<RouterLink>`（またはプログラム遷移 `router.push`）を使う。外部リンクだけ `<a>`。
- **Nuxt なのに Vue Router を手で設定**：**Nuxt はファイルベースのルーティングを内蔵**しているため、`createRouter` を自前で書く必要は基本ない。`pages/` にファイルを置くだけでルートになる。Nuxt 環境ではこのファイルの設定は不要 → [../../nuxt/nuxt3/](../../nuxt/nuxt3/)。
- **params は文字列**：`route.params.id` は常に文字列。数値として使うなら `Number(...)` 変換が要る。
- **ガードで永遠にリダイレクト**：`beforeEach` の中で常にリダイレクト先へ飛ばすと無限ループになる。「すでに遷移先なら通す」条件を入れる。
- **同一コンポーネント遷移で再取得されない**：パラメータだけ変わる遷移ではコンポーネントが再生成されないため `onMounted` が再実行されない。`watch` でパラメータを監視する。→ [lifecycle.md](./lifecycle.md)
- **history モードでリロードすると404**：`createWebHistory` はサーバ側で「全パスを index.html に返す」設定が必要。SPA ホスティングの定番設定漏れに注意。

## 関連
[pinia.md](./pinia.md) / [lifecycle.md](./lifecycle.md) / [computed_watch.md](./computed_watch.md)
