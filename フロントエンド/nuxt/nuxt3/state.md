# 状態（useState / Pinia）（Nuxt 3）

## ひとことで言うと
複数のコンポーネントで**同じデータを共有するための仕組み**。Nuxt組込の **`useState`**（SSR安全な軽量グローバル状態）と、より複雑な状態管理向けの **Pinia**（ストアライブラリ）の2択が基本。

## 役割・なぜ必要か
- ログインユーザー情報・テーマ設定・カート内容など「**ページをまたいで持ち回りたい値**」をどこに置くか、の答え。
- SSRでは**1台のサーバが複数ユーザーのリクエストを同時にさばく**。素朴に「モジュールスコープの `ref` をグローバルに共有」すると、その `ref` は全リクエストで**共有**され、Aさんのデータが Bさんに見えてしまう（**状態汚染／クロスリクエストリーク**）。
- `useState` は **リクエストごとに隔離された状態**を提供し、かつサーバで作った値をクライアントへハイドレーションで正しく引き継ぐ。だから Nuxt では「共有状態は `useState`」が原則。

## 基本の書き方（コード）
```ts
// useState：第1引数のキーで状態を一意に識別、第2引数は初期値を返す関数
const counter = useState<number>('counter', () => 0)
counter.value++ // ref と同じく .value で読み書き
```

```vue
<!-- どのコンポーネントから呼んでも同じキーなら同じ状態を共有する -->
<script setup lang="ts">
const user = useState<User | null>('auth-user', () => null)
function login(u: User) {
  user.value = u // 別コンポーネントでも 'auth-user' で同じ値が見える
}
</script>

<template>
  <p v-if="user">こんにちは {{ user.name }} さん</p>
</template>
```

```ts
// 再利用しやすいよう composable にまとめるのが定番
// composables/useCounter.ts
export const useCounter = () => useState<number>('counter', () => 0)
```

```ts
// 複雑な状態は Pinia（stores/auth.ts）
import { defineStore } from 'pinia'

export const useAuthStore = defineStore('auth', () => {
  const user = ref<User | null>(null)
  const isLoggedIn = computed(() => user.value !== null) // 派生値
  function setUser(u: User) { user.value = u }
  return { user, isLoggedIn, setUser }
})
```

## 実務での使い方・定番パターン
- **小さい共有値は `useState`**。キーは衝突しないよう `'auth-user'` のように意味のある一意な名前にする。
- **`useState` を composable で包む**（上記 `useCounter`）と、呼び出し側はキーを意識せず使えて型も一元化できる。→ [composables.md](./composables.md)
- **複雑になったら Pinia**：getter（派生値）・複数アクション・ストア間連携・devtools が要るならこちら。`@pinia/nuxt` を入れると `stores/` が自動で使える。
- **サーバ状態（API取得結果）はストアに溜め込みすぎない**。取得・キャッシュは `useFetch` / `useAsyncData` に任せ、`useState`/Pinia は「アプリのUI状態・セッション状態」に絞ると整理しやすい。→ [data_fetching.md](./data_fetching.md)
- 初期値が重い計算なら `useState('k', () => heavyInit())` のように**関数で渡す**（毎回評価しない）。

## ハマりどころ / アンチパターン
- **SSRで `ref`/オブジェクトをモジュールスコープでグローバル共有しない**。`export const store = reactive({...})` をモジュールトップに置くと全リクエストで共有され、ユーザー間で状態が漏れる。**必ず `useState`（または Pinia）を使う**。
- **`useState` のキー省略・重複に注意**。同じキーは同じ状態を指す＝意図せず別機能と衝突する。キーは明示し、ユニークに。
- **`useState` は setup 文脈で呼ぶ**：`<script setup>`・composable・プラグイン・ミドルウェア内など Nuxt のコンテキストがある場所で。イベントハンドラの奥や非同期コールバックの後（await でコンテキストが切れた後）の初回呼び出しは避ける。
- **値はシリアライズ可能に**。SSR→クライアントへJSONで渡るため、関数やクラスインスタンス・循環参照を `useState` に入れると壊れる。
- **`localStorage` を初期値にいきなり使わない**。サーバには `window` が無い。クライアント限定処理は `onMounted` や `import.meta.client` ガードで。
- **何でも Pinia に入れて肥大化**させない。サーバ取得結果は `useFetch` のキャッシュに任せ、二重管理を避ける（派生値は `computed` で）。

## 関連
[composables.md](./composables.md) / [data_fetching.md](./data_fetching.md) / [config.md](./config.md)
