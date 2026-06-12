# 状態管理（Pinia）（Vue 3）

## ひとことで言うと
複数コンポーネントで共有したい状態を、**1か所（store）にまとめて置く Vue 公式の状態管理ライブラリ**。Vuex の後継で、`defineStore` で store を定義し、どこからでも `useXxxStore()` で取り出して読み書きする。

## 役割・なぜ必要か
- props で親→子へ値を渡し、emit で子→親へ返す…という**バケツリレー（prop drilling）**は、共有範囲が広がると破綻する。Pinia は「**離れたコンポーネント間で同じ状態を共有**」するための置き場を提供する。→ [props_emit.md](./props_emit.md)
- ログインユーザー、カート、UI設定など「**アプリ全体で1つだけ持ちたい状態**」を集約し、変更の入口（actions）を限定することで**追跡・テストしやすく**する。
- Pinia は Composition API と相性がよく、store 自体が `ref` / `computed` で組まれているので**型推論が効き、`<script setup>` から自然に使える**。→ [composition_api.md](./composition_api.md)

## 基本の書き方（コード）
```ts
// stores/counter.ts — Option Stores 形式
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  // state: 関数で初期値を返す（リアクティブな素のデータ）
  state: () => ({ count: 0, name: 'Eduardo' }),

  // getters: state から導出する算出値（computed 相当）
  getters: {
    double: (state) => state.count * 2,
  },

  // actions: state を変更する手続き（同期/非同期どちらも可）
  actions: {
    increment() { this.count++ },
    async fetchAndSet() {
      const res = await fetch('/api/count')
      this.count = (await res.json()).count
    },
  },
})
```
```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCounterStore } from '@/stores/counter'

const store = useCounterStore()
// ★ state / getters は storeToRefs で取り出す（リアクティブ維持）
const { count, double } = storeToRefs(store)
// ★ actions はメソッドなので直接分割代入でOK
const { increment } = store
</script>

<template>
  <button @click="increment">count: {{ count }} / double: {{ double }}</button>
</template>
```

## 実務での使い方・定番パターン
- **store はアプリ起動時に登録**：`main.ts` で `app.use(createPinia())` してから `app.mount(...)`。
- **`useXxxStore()` は setup 内で呼ぶ**：Pinia が有効化された後に呼ぶ必要があるため、コンポーネントの `<script setup>`（または他の store / composable）内で取得する。
- **読み取りは `storeToRefs`、変更は actions 経由**：直接 `store.count++` も動くが、変更経路を actions にそろえると意図と副作用が追いやすい。→ [composition_api.md](./composition_api.md)
- **Setup Stores 形式**も使える：`defineStore('x', () => { const count = ref(0); ... return { count } })`。`ref`=state、`computed`=getter、関数=action という対応。Composition API そのままで書けるので好みで選ぶ。
- **store の分割**：機能ごとに `useUserStore` / `useCartStore` のように分ける。store から別の store を呼んで合成もできる。

## ハマりどころ / アンチパターン
- **store を分割代入してリアクティブが切れる**（最頻出）：`const { count } = useStore()` だと、ただの値のコピーになり、以後 store が変わっても画面が**更新されない**。state/getters は必ず **`const { count } = storeToRefs(store)`** で取り出す。
  - 逆に **actions（関数）を `storeToRefs` に通すと壊れる**。actions は `const { increment } = store` のように直接取り出す。
- **`useXxxStore()` をモジュールのトップレベルで呼ぶ**：Pinia 有効化前に実行され `getActivePinia` エラーになりやすい。setup 内・関数内で呼ぶ。
- **サーバ状態を Pinia に丸ごと入れて手動同期する**：API から取ったデータのキャッシュ・再取得・ローディング管理を store で自作すると複雑化する。サーバ状態は**別管理**が定石（Nuxt なら `useFetch` / `useAsyncData`、SPA なら TanStack Query など）。Pinia は**クライアント状態**に寄せる。→ [../../nuxt/nuxt3/](../../nuxt/nuxt3/)
- **state を外から直接ミューテートしまくる**：どこで何が変わったか追えなくなる。変更は actions に集約する。
- **巨大な単一 store**：1つに全部詰めると依存が絡む。機能・ドメイン単位で小さく分ける。

## 関連
[composition_api.md](./composition_api.md) / [props_emit.md](./props_emit.md) / [routing.md](./routing.md)
