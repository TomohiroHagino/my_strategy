# computed / watch（Vue 3）

## ひとことで言うと
- **`computed`**：他のリアクティブな状態（`ref` / `reactive`）から**派生する値**を作る。結果は**キャッシュ**され、依存が変わったときだけ再計算される。基本は **getter のみ**（読み取り専用）。
- **`watch`**：**特定の値の変化を監視**し、変わったときに**副作用**（API呼び出し・ログ・ルータ遷移など）を走らせる。
- **`watchEffect`**：関数内で**使った依存を自動追跡**し、変化のたびに再実行する。ソースを明示しない手軽版。

## 役割・なぜ必要か
- **「値が欲しい」のか「処理を起こしたい」のか**で道具が分かれる。表示用の派生値は `computed`、変化に応じた手続きは `watch` / `watchEffect`。
- `computed` はキャッシュされるため、テンプレートで何度参照しても再計算は1回。重い計算ほど効く。
- `watch` は「いつ・何が変わったか（新旧値）」を受け取れるので、**条件付きの副作用**を書きやすい。

## 基本の書き方（コード）
```vue
<script setup lang="ts">
import { ref, computed, watch, watchEffect } from 'vue'

const firstName = ref('Taro')
const lastName  = ref('Yamada')

// computed: 依存から派生する値（キャッシュされる）
const fullName = computed(() => `${firstName.value} ${lastName.value}`)

// 書き込み可能 computed（getter + setter）
const editableName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (v: string) => {
    const [f, l] = v.split(' ')
    firstName.value = f
    lastName.value  = l ?? ''
  },
})

// watch: 特定の値の変化に反応して副作用
const userId = ref(1)
watch(userId, async (newId, oldId) => {
  console.log(`id ${oldId} -> ${newId}`)
  // await fetchUser(newId)
})

// watchEffect: 使った依存を自動追跡（初回も即実行）
watchEffect(() => {
  console.log(`now: ${firstName.value} ${lastName.value}`)
})
</script>

<template>
  <p>{{ fullName }}</p>
  <input v-model="editableName" />
</template>
```

## 実務での使い方・定番パターン
- **表示整形・絞り込み・合計** は `computed`。例：`const visible = computed(() => list.value.filter(x => x.active))`。
- **検索ボックス → API**：`watch(query, debouncedFetch)` のように、入力変化を副作用へつなぐ。
- **ソースの指定方法**（`watch` の第1引数）：
  - `ref`：そのまま渡す → `watch(userId, fn)`
  - **getter 関数**：`reactive` の一部や複数値 → `watch(() => state.count, fn)`
  - **配列**：複数監視 → `watch([a, b], ([na, nb]) => {})`
- **`immediate: true`**：登録直後にも1回実行（初期ロード兼用）。`watch(src, fn, { immediate: true })`。
- **`deep: true`**：オブジェクト/配列の**内部変化**まで監視。`reactive` は既定で deep、`ref(object)` は明示が必要なことがある。
- 後始末が要る監視は `watchEffect` の `onCleanup` や、`watch` 内でのクリーンアップで処理する。

## ハマりどころ / アンチパターン
- **`computed` の中で副作用を書かない**（API呼び出し・`ref` の書き換え・ログ送信など）。`computed` は「純粋に値を返す」場所。副作用は `watch` / `watchEffect` へ。
- **派生値なのに `watch` で別の `ref` に詰め直す**のは冗長。`fullName = a + b` のような派生は `computed` 一択（同期ズレ・二重管理を防ぐ）。
- **`computed` vs `methods`**：メソッドは呼ぶたびに毎回実行され**キャッシュされない**。テンプレートで何度も使う派生値はメソッドより `computed` が有利。
- **`watch` のソース指定ミス**：`reactive` の中身を監視したいのに値だけ渡すと追跡されない。`() => state.x` のように **getter で渡す**。
- **`deep` の付け忘れ**：オブジェクトの中身を変えても発火しない、という典型。必要なら `{ deep: true }`。
- **`immediate` の付け忘れ**：「初回ロードが走らない」原因の定番。初期実行が要るなら明示。
- **`watchEffect` の追跡漏れ**：条件分岐で初回に通らなかった `ref` は依存に登録されない。確実に監視したい値は `watch` でソース明示が安全。

## 関連
[reactivity.md](./reactivity.md) / [composition_api.md](./composition_api.md)
