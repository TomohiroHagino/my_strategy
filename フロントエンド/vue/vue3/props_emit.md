# props / emit / コンポーネントの v-model（Vue 3）

## ひとことで言うと
- **`defineProps`**：**親 → 子**へ渡すデータを、子側で**型付き**に受け取る宣言。
- **`defineEmits`**：**子 → 親**へ**イベント**を投げる宣言。子の中の出来事を親に通知する。
- **コンポーネントの `v-model`**：親子で値を**双方向に同期**する仕組み。Vue 3.4+ の **`defineModel`**、または **`modelValue` props ＋ `update:modelValue` イベント**で実現する。

## 役割・なぜ必要か
- データは「上から下（props）」、通知は「下から上（emit）」という**一方向データフロー**を守ることで、状態の出どころが明確になりデバッグしやすい。
- フォーム部品（入力欄・チェックボックス等）は「親の状態を表示しつつ、変更を親に返す」が必要。これを簡潔に書くのが `v-model`。

## 基本の書き方（コード）
```vue
<!-- 子：Child.vue -->
<script setup lang="ts">
// props：親から受ける（型付き・既定値つき）
const props = withDefaults(
  defineProps<{ title: string; count?: number }>(),
  { count: 0 },
)

// emits：親へイベントを投げる
const emit = defineEmits<{
  (e: 'submit', payload: string): void
  (e: 'close'): void
}>()

const onClick = () => emit('submit', props.title)
</script>

<template>
  <h3>{{ title }} ({{ count }})</h3>
  <button @click="onClick">送信</button>
  <button @click="emit('close')">閉じる</button>
</template>
```

```vue
<!-- 親 -->
<script setup lang="ts">
import Child from './Child.vue'
const onSubmit = (v: string) => console.log('submit', v)
</script>

<template>
  <Child title="見出し" :count="3" @submit="onSubmit" @close="() => {}" />
</template>
```

```vue
<!-- v-model 対応の子（Vue 3.4+：defineModel が最も簡潔） -->
<script setup lang="ts">
const model = defineModel<string>() // 親の v-model と双方向同期
</script>
<template>
  <input :value="model" @input="model = ($event.target as HTMLInputElement).value" />
</template>

<!-- 親：<MyInput v-model="text" /> だけで繋がる -->
```

```vue
<!-- v-model の中身（3.4 未満や仕組み理解用）：modelValue + update:modelValue -->
<script setup lang="ts">
defineProps<{ modelValue: string }>()
const emit = defineEmits<{ (e: 'update:modelValue', v: string): void }>()
</script>
<template>
  <input :value="modelValue"
         @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)" />
</template>
```

## 実務での使い方・定番パターン
- **型付き props**：`defineProps<{...}>()` でTSの型を直接付ける。既定値は `withDefaults` で。
- **イベント命名**：`@submit` / `@update:xxx` のように動詞・対象で。emit名は型で縛ると安全。
- **`v-model` の引数違い**：`v-model:title` のように名前付きにでき、子では `defineModel('title')`（または `title` / `update:title`）で受ける。複数 `v-model` も可能。
- **`defineModel` を優先**（3.4+）：ボイラープレートが激減する。古い環境なら `modelValue` ＋ `update:modelValue` を手書き。

## ハマりどころ / アンチパターン
- **props は読み取り専用（直接変更しない）**：子で `props.count++` のように書き換えてはいけない。変更したいなら**emit で親に依頼**するか、ローカルの `ref` にコピーして扱う。直接変更は一方向フローを壊し、警告も出る。
- **オブジェクト/配列 props の中身いじり**：参照渡しなので技術的には変わるが、これも禁止筋。親の状態を勝手に書き換えると追跡不能になる。
- **`v-model` の仕組み誤解**：`v-model="x"` は実体として `:modelValue="x" @update:modelValue="x = $event"` の糖衣。子が `update:modelValue` を**emitしないと親に反映されない**。
- **`defineModel` の戻り値を破壊**：`defineModel()` が返すのはリアクティブな ref。`.value` 書き換え or テンプレ代入で同期する。別変数に詰め替えると同期が切れる。
- **emit名のタイポ**：親の `@submit` と子の `emit('sbumit')` が食い違っても静かに何も起きない。型定義で防ぐ。

## 関連
[components.md](./components.md) / [composition_api.md](./composition_api.md) / [reactivity.md](./reactivity.md)
