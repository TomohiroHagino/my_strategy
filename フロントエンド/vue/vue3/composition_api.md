# Composition API（`<script setup>`）（Vue 3）

## ひとことで言うと
コンポーネントのロジックを **関数的に組み立てる API**。状態・算出値・監視・ライフサイクルを `ref` / `computed` / `watch` / `onMounted` などの**関数として**並べて書く。**`<script setup>` が現在の標準的な書き方**で、定型コードが最も少ない。

## 役割・なぜ必要か
- Options API（`data` / `methods` / `computed` をオプション**オブジェクト**で分けて書く方式）では、1つの関心事（例：検索機能）が `data`・`methods`・`watch`…と**バラバラの場所に散る**。
- Composition API は**関心事ごとにまとめて書ける**ので、大きなコンポーネントでも追いやすい。
- ロジックを**関数として切り出して再利用**できる（= **composable**）。Options API のmixinより衝突が少なく明示的。

## 基本の書き方（コード）
```vue
<!-- 標準：<script setup>。トップレベルがそのままコンポーネントスコープ -->
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

const count = ref(0)                          // 状態
const doubled = computed(() => count.value * 2) // 算出値
const increment = () => { count.value++ }       // メソッド

onMounted(() => console.log('mounted'))         // ライフサイクル
</script>

<template>
  <button @click="increment">{{ count }} / {{ doubled }}</button>
</template>
```

```ts
// composable：ロジックを関数に切り出して再利用（useXxx 命名）
// composables/useCounter.ts
import { ref, computed } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const doubled = computed(() => count.value * 2)
  const increment = () => { count.value++ }
  return { count, doubled, increment } // 必要なものを公開
}
```

```vue
<script setup lang="ts">
import { useCounter } from '@/composables/useCounter'
const { count, doubled, increment } = useCounter(10) // どのコンポーネントでも使える
</script>
```

## 実務での使い方・定番パターン
- **`<script setup>` を既定にする**：`return` 不要で、宣言した変数・関数はそのままテンプレートから使える。`defineProps` / `defineEmits` も使える。→ [props_emit.md](./props_emit.md)
- **composable（`useXxx`）でロジック共有**：`useFetch` / `useMouse` / `useForm` のように、状態＋手続きをセットで返す。返り値は分割代入で受ける。
- **関心事ごとにまとめる**：「検索」関連の `ref` / `computed` / `watch` を近くに置く。Optionsのように種類で分けない。
- **明示的な `setup()`**：`<script setup>` を使わない場合は `setup(props, ctx)` 内でロジックを書き、`return` で公開する（古い書き方だが理解しておくと読める）。
- Options API も**まだ使える**ので、既存コードはOptionsのまま残ることも多い。新規は Composition が無難。

## ハマりどころ / アンチパターン
- **Options と Composition の混同**：`<script setup>` の中で `this.xxx` は使えない（`this` がコンポーネントインスタンスを指さない）。`data() {}` 等のオプションも併用しない。どちらかに寄せる。
- **`<script setup>` のトップレベルがコンポーネントスコープ**：そこで宣言したものはインスタンス単位で**毎回新規に作られる**。モジュールのトップレベル（`<script setup>` の外）に書いた値は**全インスタンスで共有**されてしまうので注意。
- **composable をコンポーネント外/条件分岐内で呼ぶ**：`useXxx` は `<script setup>` のトップレベルで呼ぶのが基本（ライフサイクル登録が絡むため）。
- **リアクティビティの取り違え**：composable から返した `ref` を分割代入しても `.value` 経由ならリアクティブを保てるが、`reactive` を分割代入すると切れる。→ [reactivity.md](./reactivity.md)
- **何でも composable 化**：再利用しない単発ロジックまで切り出すと逆に追いにくい。共有・テストの必要が出てから抽出する。

## 関連
[reactivity.md](./reactivity.md) / [computed_watch.md](./computed_watch.md) / [props_emit.md](./props_emit.md)
