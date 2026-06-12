# ライフサイクル（onMounted 等）（Vue 3）

## ひとことで言うと
コンポーネントが「生成 → DOMにマウント → 更新 → 破棄」と移り変わる各タイミングに**自分の処理を差し込むためのフック関数**。Composition API では `onMounted` / `onUnmounted` などを `setup`（＝`<script setup>`）内で呼んで登録する。

## 役割・なぜ必要か
- テンプレートを描いて画面に出す Vue の処理の「**節目**」に介入できる。たとえば「DOMが出来上がった後に要素のサイズを測る」「画面に出た直後にAPIを叩く」「消えるときにタイマーを止める」など、**タイミングが命の処理**を正しい場所に置ける。
- Composition API では Options API の `mounted() {}` のような**オプション名ではなく関数**で登録する。これにより、関連するロジック（取得・購読・解除）を1つの関数（composable）に**まとめて再利用**できる。→ [composition_api.md](./composition_api.md)
- 「いつ動くか」を Vue 任せにせず明示できるので、副作用（DOM操作・通信・購読）の**開始と後始末を対にして書ける**のが本質的な価値。

## 基本の書き方（コード）
```vue
<script setup lang="ts">
import { ref, onBeforeMount, onMounted, onUpdated, onUnmounted } from 'vue'

const count = ref(0)
const el = ref<HTMLElement | null>(null)   // template ref

onBeforeMount(() => {
  // DOMはまだ無い。初期化（軽い準備）程度に。
  console.log('before mount')
})

onMounted(() => {
  // ここで初めて DOM が存在する。要素アクセス・データ取得はここ。
  console.log('mounted', el.value?.offsetWidth)
  // 例: リスナ登録（解除と必ず対にする）
  const onResize = () => { /* ... */ }
  window.addEventListener('resize', onResize)

  // onUnmounted を mounted の中で登録してもよい（解除を近くに書ける）
  onUnmounted(() => window.removeEventListener('resize', onResize))
})

onUpdated(() => {
  // リアクティブな変化でDOMが再描画された“後”。多用は注意（後述）。
  console.log('updated', count.value)
})

onUnmounted(() => {
  // 破棄時の後始末（タイマー・購読・WebSocket等の解除）
  console.log('unmounted')
})
</script>

<template>
  <div ref="el">
    <button @click="count++">count: {{ count }}</button>
  </div>
</template>
```

## 実務での使い方・定番パターン
- **DOM操作・サイズ計測・フォーカス**は `onMounted`。`setup` 直下では DOM がまだ無いので測れない。`ref` で取った要素は `onMounted` 内で `el.value` が使える。
- **データ取得（fetch）**も基本 `onMounted`。SSR を使うなら「サーバでも実行されるか」を意識する（Nuxt では `onMounted` はクライアントのみ実行）。→ サーバ取得は [../../nuxt/nuxt3/](../../nuxt/nuxt3/) の `useFetch` 系へ。
- **リスナ・タイマー・購読の解除**は `onUnmounted`。`addEventListener` ↔ `removeEventListener`、`setInterval` ↔ `clearInterval` を**必ず対**で書く。解除を `onMounted` の中で登録すると、登録と解除が近くに並んで漏れにくい。
- **エラー捕捉**は `onErrorCaptured`、**keep-alive** では `onActivated` / `onDeactivated` を使う。
- 取得・購読・解除を**1つの composable** にまとめると再利用できる（`useMousePosition()` 等）。フック呼び出しを composable 内に隠せるのが Composition API の強み。→ [composition_api.md](./composition_api.md)

```ts
// composables/useNow.ts — タイマーを安全にカプセル化
import { ref, onMounted, onUnmounted } from 'vue'
export function useNow() {
  const now = ref(Date.now())
  let id: number
  onMounted(() => { id = window.setInterval(() => (now.value = Date.now()), 1000) })
  onUnmounted(() => clearInterval(id))   // 後始末まで内包
  return { now }
}
```

## ハマりどころ / アンチパターン
- **フックを非同期や条件分岐の中で登録する**：`onMounted` などは `setup`（`<script setup>`直下）で**同期的**に呼ぶ必要がある。`await` の後や `if` の中、コールバック内で呼ぶと登録されず動かない。
- **クリーンアップ忘れ（メモリリーク）**：`addEventListener` / `setInterval` / WebSocket 購読を `onUnmounted` で解除しないと、コンポーネントが消えても処理が生き続け、リークやエラーの原因になる。
- **`onMounted` でしか触れない DOM を `setup` 直下で触る**：マウント前は DOM が無いので `el.value` は `null`。計測・フォーカスは `onMounted` 以降で。
- **`onUpdated` でリアクティブな state を変更**：再描画 → `onUpdated` → 変更 → 再描画…の無限ループになりやすい。値の変化に反応したいなら `onUpdated` ではなく `watch` を使う。→ [computed_watch.md](./computed_watch.md)
- **データ取得を `onBeforeMount` に置く**：早く見えて利点は薄く、DOM 前提の処理と混ざりやすい。取得は `onMounted` に寄せるのが無難。
- **Options API と混在させて二重登録**：`mounted()` と `onMounted()` を同じ意図で両方書くと処理が二度走る。どちらかに統一する。

## 関連
[composition_api.md](./composition_api.md) / [reactivity.md](./reactivity.md) / [computed_watch.md](./computed_watch.md)
