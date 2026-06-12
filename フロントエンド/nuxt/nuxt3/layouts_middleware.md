# レイアウト / ルートミドルウェア（Nuxt 3）

## ひとことで言うと
**レイアウト** = 複数ページで共通の「ガワ」（ヘッダ・サイドバー等）をまとめる枠。
**ルートミドルウェア** = ページ遷移の**前に**走らせる関数で、認証ガードやリダイレクトの定番置き場。

## 役割・なぜ必要か
- レイアウトは「ページごとに中身は違うが、外側は同じ」を**DRY**に保つための仕組み。各ページで毎回ヘッダを書かない。`layouts/default.vue` を置けば、`app.vue` の `<NuxtLayout>` が自動でこれを使う。
- ミドルウェアは「**このページを描画してよいか**」の判断を遷移前に挟む層。コンポーネント内で `if (!user) redirect()` を散らすより、入口で一括ガードできる。
- グローバルミドルウェア（`.global.ts`）は**全ページ遷移で必ず**走るので、横断ログ・全体ガードに向く。名前付きは `definePageMeta` で**明示したページだけ**走る。

## 基本の書き方（コード）
```vue
<!-- layouts/default.vue : 共通レイアウト。<slot/> が必須 -->
<template>
  <div class="app-shell">
    <header><nav aria-label="Main"><NuxtLink to="/">Home</NuxtLink></nav></header>
    <main>
      <slot /> <!-- ← ページ本体がここに差し込まれる。これが無いと中身が消える -->
    </main>
    <footer>© 2026</footer>
  </div>
</template>
```

```vue
<!-- layouts/admin.vue : 別レイアウトも作れる -->
<template>
  <div class="admin"><aside>管理メニュー</aside><slot /></div>
</template>
```

```vue
<!-- pages/dashboard.vue : ページ側でレイアウトとミドルウェアを指定 -->
<script setup lang="ts">
definePageMeta({
  layout: 'admin',        // layouts/admin.vue を使う（既定は 'default'）
  middleware: 'auth',     // middleware/auth.ts を遷移前に実行
})
</script>
<template><h1>ダッシュボード</h1></template>
```

```ts
// middleware/auth.ts : 名前付きミドルウェア（認証ガード）
export default defineNuxtRouteMiddleware((to, from) => {
  const { loggedIn } = useUserSession() // 自前 composable 等
  if (!loggedIn.value) {
    // 未ログインならログインページへ。元の行先を query で覚えておく
    return navigateTo({ path: '/login', query: { redirect: to.fullPath } })
  }
})
```

```ts
// middleware/log.global.ts : 全遷移で走るグローバル版（.global で自動登録）
export default defineNuxtRouteMiddleware((to) => {
  console.log('[nav]', to.path) // サーバ/クライアント両方で実行される点に注意
})
```

## 実務での使い方・定番パターン
- **認証ガード**：保護ページに `middleware: 'auth'` を付け、未ログインは `navigateTo('/login')`。`to.fullPath` を `redirect` に保存してログイン後に戻す。
- **遷移をやめる**：条件を満たさないときは `abortNavigation()` で遷移自体を中断（403相当の表示を出す等）。引数にエラーを渡すとエラーページへ。
- **レイアウト動的切替**：`setPageLayout('admin')` でランタイムに変更可能。ログイン状態でガワを切り替えたい時に。
- **レイアウトなしページ**：`definePageMeta({ layout: false })` でラップを外す（埋め込み用ページ等）。
- **名前付き vs グローバル**：個別ページのガードは名前付き、全体の計測・共通リダイレクトはグローバル。グローバルは順序を `01.first.global.ts` のように接頭辞で制御できる。
- **`navigateTo` は必ず return**：ミドルウェア内では `return navigateTo(...)` とする。`await navigateTo()` でも可だが、返り値を返さないとガードが効かないことがある。

## ハマりどころ / アンチパターン
- **`<slot/>` 忘れ**：レイアウトに `<slot/>` が無いとページ本体が描画されず**画面が真っ白／ガワだけ**になる。最頻出ミス。
- **ミドルウェアはサーバ/クライアント両方で走る**：SSR初回はサーバ、以降のSPA遷移はクライアント。`window`/`localStorage` を無条件に触ると**サーバ側で落ちる**。`import.meta.client` でガードするか、Cookie 由来の状態（`useCookie`）を使う。
- **`navigateTo` を呼ぶだけで return しない**：リダイレクトが効かず、保護ページがそのまま描画される。必ず `return`。
- **ミドルウェアで重い非同期処理**：遷移ごとに毎回走るため体感が遅くなる。重い取得はページ側 `useFetch` へ寄せる。→ [data_fetching.md](./data_fetching.md)
- **`.global` を付け忘れ**：ファイルを `middleware/` に置いただけでは自動実行されない。全ページ実行したいなら `xxx.global.ts`、個別なら `definePageMeta` で参照。
- **`definePageMeta` をマクロとして外で使う**：コンパイル時マクロなので `<script setup>` 直下に静的に書く。変数で組み立てると動かない。

## 関連
[pages_routing.md](./pages_routing.md) / [data_fetching.md](./data_fetching.md) / [state.md](./state.md)
