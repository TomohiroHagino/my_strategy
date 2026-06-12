# コンポーザブル（composables/）（Nuxt 3）

## ひとことで言うと
`composables/` ディレクトリに置いた **`useXxx` という関数で、ロジックを再利用するための部品**。Vue の Composition API（`ref`/`computed`/ライフサイクル等）を組み合わせた関数で、Nuxt がプロジェクト全体で **import 不要（自動インポート）** にしてくれる。

## 役割・なぜ必要か
- 「複数コンポーネントで同じロジック（カウンタ・認証・スクロール検知…）を使い回したい」を、継承やコピペでなく**関数の合成**で解決する。
- Vue の「composition 関数」＝状態とふるまいを `ref`/`computed`/`watch` でまとめて返す関数。これを `composables/` に置くと、Nuxt が**ファイル名・export名から自動でグローバル利用可能**にする（`import` を書かなくてよい）。
- Nuxt 自身も多くの組込 composable を提供：`useRoute` / `useRouter` / `useRuntimeConfig` / `useState` / `useFetch` / `useAsyncData` / `useHead` など。自作のものも同じ流儀で並ぶ。

## 基本の書き方（コード）
```ts
// composables/useCounter.ts  → どこでも useCounter() で使える
export const useCounter = (initial = 0) => {
  const count = ref(initial)
  const inc = () => count.value++
  const dec = () => count.value--
  const isZero = computed(() => count.value === 0)
  return { count, inc, dec, isZero } // 状態とふるまいをまとめて返す
}
```

```vue
<script setup lang="ts">
// import 不要：Nuxt が自動インポート
const { count, inc, isZero } = useCounter(10)
</script>

<template>
  <button @click="inc">count = {{ count }}（zero? {{ isZero }}）</button>
</template>
```

```ts
// 組込 composable を内部で使い、共有状態を包む例
// composables/useAuth.ts
export const useAuth = () => {
  const user = useState<User | null>('auth-user', () => null) // SSR安全な共有状態
  const config = useRuntimeConfig()                           // 設定の参照
  const isLoggedIn = computed(() => user.value !== null)
  async function login(email: string, password: string) {
    user.value = await $fetch('/api/login', {
      method: 'POST',
      body: { email, password, appId: config.public.appId },
    })
  }
  return { user, isLoggedIn, login }
}
```

## 実務での使い方・定番パターン
- **1ファイル1 composable**を基本に、`use` 始まりの名前で `composables/` 直下へ。サブディレクトリは（既定では）トップレベルのみ自動インポート対象なので、深い階層は `index.ts` で再エクスポートする。
- **状態共有は `useState` を中に包む**（上記 `useAuth`）。これで「ロジック再利用＋SSR安全な共有状態」を1つの `useXxx` に集約できる。→ [state.md](./state.md)
- **データ取得 composable**：`useFetch`/`useAsyncData` をラップして、エンドポイントや変換を共通化する。→ [data_fetching.md](./data_fetching.md)
- **副作用の後始末**：`onMounted` で登録したリスナを `onScopeDispose`/`onUnmounted` で外す、を composable 内に閉じ込めると呼び出し側が楽。
- コンポーネントとの境界：UIに依存しない純ロジックは composable、見た目は `components/`。→ [components_autoimport.md](./components_autoimport.md)

## ハマりどころ / アンチパターン
- **ライフサイクル系・`useState`・`useRoute` 等は setup 文脈で呼ぶ**。composable を「`<script setup>`・別 composable・プラグイン・ミドルウェアのトップレベル」で呼ぶのは可。だが**イベントハンドラの中や `await` をまたいだ後**の初回呼び出しは Nuxt コンテキストが切れて警告/失敗になりがち。`onMounted(() => useXxx())` のような遅延呼びは避け、setup で取得して中の関数だけ後で呼ぶ。
- **命名規則を守る**：自動インポートは `use` 始まりの関数を前提にした慣習。`getCounter` のような名前だと意図が伝わらず、`use` 始まりでないと自動インポート対象として扱われないケースもある。
- **composable 内でモジュールスコープに `ref` を置いて共有しない**（SSRで状態汚染）。共有したいなら `useState`。→ [state.md](./state.md)
- **巨大化させない**：1つの `useXxx` に責務を詰め込みすぎたら、小さな composable に分割して合成する（高凝集・低結合）。
- **クライアント専用APIに注意**：`window`/`localStorage` を使う composable は SSR で落ちる。`import.meta.client` ガードや `onMounted` 内で扱う。
- **`$fetch` と `useFetch` の混同**：composable のトップで生データが欲しいだけなら `$fetch`、SSR連携・キャッシュ・refresh が要るなら `useFetch`。用途で使い分ける。

## 関連
[components_autoimport.md](./components_autoimport.md) / [state.md](./state.md) / [data_fetching.md](./data_fetching.md)
