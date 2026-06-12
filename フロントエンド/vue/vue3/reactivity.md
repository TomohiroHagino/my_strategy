# リアクティビティ（Reactivity）（Vue 3）

## ひとことで言うと
状態（データ）を**追跡可能なリアクティブ値**にして、「その値を変えると、それを使うテンプレート・`computed`・`watch` が自動で再計算・再描画される」仕組み。中心は `ref` と `reactive`。

## 役割・なぜ必要か
- Vueの核そのもの。**状態を変える → 画面が追従する**を成立させるのがリアクティビティ。手動でDOM更新を書かなくてよい。
- Vue 3 は Proxy ベースで実装され、`ref`（プリミティブ向けのラッパ）と `reactive`（オブジェクト向けのProxy）の2系統を提供する。
- 「**いつ `.value` が要るか**」「**いつリアクティブ性が切れるか**」を理解しないとバグの温床になる。→ [computed_watch.md](./computed_watch.md) / [composition_api.md](./composition_api.md)

## 基本の書き方（コード）
```vue
<script setup lang="ts">
import { ref, reactive, toRefs } from 'vue'

// ref：プリミティブ向け。script内は .value でアクセス／代入する
const count = ref(0)
console.log(count.value) // 0
count.value++            // ← 再代入も .value 経由

// ref はオブジェクトも包める（内部的に reactive 化される）
const user = ref({ name: 'Taro', age: 20 })
user.value.age = 21      // .value 経由でプロパティ操作

// reactive：オブジェクト向け。.value は不要だが、再代入・分割代入に弱い
const state = reactive({ items: [] as string[], loading: false })
state.loading = true     // ← そのままプロパティ操作（.value 不要）

// toRefs：reactive を分割しても各プロパティのリアクティブ性を保つ
const { loading } = toRefs(state)
console.log(loading.value) // ref 化されるので .value で読む
</script>

<template>
  <!-- テンプレートでは ref は自動アンラップ（.value 不要） -->
  <p>{{ count }} / {{ user.name }} / {{ state.loading }}</p>
  <button @click="count++">+1</button>
</template>
```

## 実務での使い方・定番パターン
- **基本は `ref` を使うのが安全**：プリミティブでもオブジェクトでも `ref` で統一すると、「どれがリアクティブか」が `.value` の有無で一目で分かり、再代入も `x.value = newObj` で素直にできる。迷ったら `ref`。
- **`.value` のルールを覚える**：
  - **script内（JS/TSロジック）では `.value` が必要**（読みも書きも）。
  - **テンプレート内では自動アンラップされ `.value` 不要**（`{{ count }}` でよい）。
- **`reactive` はネストした状態オブジェクトに**：フォーム全体やストア的なまとまりを1つの `reactive({...})` で持つと、各プロパティを `state.xxx` で直接操作できて `.value` が不要になり読みやすい。
- **`toRefs` / `toRef`**：`reactive` を分割代入で取り出したいとき、`toRefs(state)` を通すと各プロパティが ref になりリアクティブ性を保てる。composable の戻り値を分割代入させたい時の定番。→ [composition_api.md](./composition_api.md)
- **配列・オブジェクトの更新**：`ref` の場合 `list.value.push(x)` でOK（Vue 3はProxyなので添字代入も検知される）。`ref` 自体を差し替えるなら `list.value = [...list.value, x]`。

## ハマりどころ / アンチパターン
- **`.value` 忘れ（最頻出）**：script内で `count++` ではなく `count.value++`、`if (count)` ではなく `if (count.value)`。`.value` を付け忘れると ref オブジェクトそのものを操作・判定してしまい、値が変わらない／常に truthy になる。**テンプレートでは逆に `.value` を付けない**（自動アンラップ）。
- **`reactive` の分割代入でリアクティブ性が切れる**（要注意）：
  ```ts
  const state = reactive({ loading: false })
  const { loading } = state   // NG: loading はただの false（追跡が切れる）
  // 以後 state.loading を変えても loading は更新されない
  const { loading } = toRefs(state) // OK: ref 化されて追跡が保たれる
  ```
  `reactive` をスプレッド（`{ ...state }`）や分割代入で取り出すと、その時点の素の値になり**追跡が外れる**。分割するなら `toRefs` を通す。
- **`reactive` の丸ごと再代入は効かない**：`let state = reactive({...}); state = reactive({...})` としても、元の参照を見ているテンプレートは更新されない。**まるごと差し替えたい状態は `ref` で持つ**（`obj.value = newObj`）。
- **`ref` をテンプレートで `.value` 付きで書く**：`{{ count.value }}` は不要（自動アンラップ）。ただし**ネストして ref を別 ref の中に入れた場合**は自動アンラップされないことがあるので、状態の入れ子を浅く保つ。
- **props を直接 `reactive`/`ref` 化して書き換える**：props は読み取り専用。書き換えたいなら `computed` や `emit` 経由で親に返す。→ [props_emit.md](./props_emit.md)

## 関連
[computed_watch.md](./computed_watch.md) / [composition_api.md](./composition_api.md) / [directives.md](./directives.md)
