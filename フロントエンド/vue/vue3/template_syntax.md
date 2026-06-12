# テンプレート構文（Vue 3）

## ひとことで言うと
`<template>` 内で **状態をHTMLに反映するための書き方**。テキストは補間 `{{ }}`、属性は `v-bind`（短縮 `:`）、イベントは `v-on`（短縮 `@`）で結びつける。中に書けるのは「**式（expression）**」だけ。

## 役割・なぜ必要か
- Vue の核は「**状態(ref)を変えると、それを使うテンプレートが自動で再描画される**」こと。その「状態 ↔ DOM」の橋渡しを担うのがテンプレート構文。
- 生の JS で `element.textContent = ...` や `addEventListener(...)` を手書きする代わりに、**宣言的に**「ここにこの値を出す」「このイベントでこの処理」と書ける。
- `{{ }}`（テキスト）、`:`（属性）、`@`（イベント）の3つを押さえれば、テンプレートの大半は読める・書ける。

## 基本の書き方（コード）
```vue
<script setup lang="ts">
import { ref } from 'vue'

const message = ref('こんにちは')
const url = ref('https://vuejs.org')
const isActive = ref(true)
const color = ref('tomato')
const count = ref(0)
</script>

<template>
  <!-- 1) 補間: テキストとして埋め込む -->
  <p>{{ message }}</p>
  <p>{{ message.toUpperCase() }}</p>     <!-- 式はOK -->
  <p>{{ count + 1 }}</p>                  <!-- 計算もOK -->

  <!-- 2) 属性バインド: v-bind → 短縮 ":" -->
  <a v-bind:href="url">公式（長い書き方）</a>
  <a :href="url">公式（短縮）</a>          <!-- 実務はこちら -->

  <!-- 3) イベント: v-on → 短縮 "@" -->
  <button v-on:click="count++">+1（長い）</button>
  <button @click="count++">+1（短縮）</button>

  <!-- 4) :class バインド（オブジェクト構文: true のキーだけ付く） -->
  <div :class="{ active: isActive, 'is-large': count > 5 }">box</div>

  <!-- 5) :style バインド（オブジェクト構文。camelCase で書く） -->
  <div :style="{ color: color, fontSize: count + 'px' }">styled</div>

  <!-- 配列構文で複数まとめても良い -->
  <div :class="['card', isActive ? 'on' : 'off']">card</div>
</template>
```

## 実務での使い方・定番パターン
- **属性は基本 `:`、イベントは基本 `@`** を使う（`v-bind:` / `v-on:` のフル表記はほぼ書かない）。
- `:class` の **オブジェクト構文** `{ クラス名: 真偽値 }` が条件付きクラスの定番。静的 class と併用でき、`class="card" :class="{ active }"` のように混ぜられる。
- 複雑な算出は**テンプレートに長い式を書かず** `computed` に逃がす（可読性・再計算最適化）。→ [computed_watch.md](./computed_watch.md)
- 動的属性名 `:[key]="value"`、真偽属性（`:disabled="isBusy"` で false なら属性ごと消える）も覚えると便利。
- `v-html` で生HTMLを描画できるが **XSS の温床**なので、ユーザー入力には絶対使わない。
- ディレクティブ（`v-if` / `v-for` / `v-model` 等）は別項目。→ [directives.md](./directives.md)

## ハマりどころ / アンチパターン
- **`{{ }}` は属性値には使えない**：`<a href="{{ url }}">` は NG。属性には必ず `:` を使う → `<a :href="url">`。
- **テンプレートに書けるのは「式」だけで「文」は不可**：`{{ if (x) {...} }}` や `{{ const a = 1 }}` は書けない。条件は三項演算子 `{{ ok ? 'A' : 'B' }}` で。
- **`@click="fn()"` と `@click="fn"` の違い**：引数なしなら `@click="fn"`（参照を渡す）。`@click="fn()"` は即時呼び出し扱い。イベント引数が欲しいときは `@click="fn($event)"`。
- **`:style` のプロパティ名**：`font-size` ではなく `fontSize`（camelCase）。ケバブケブで書くならクォート必須 `'font-size'`。
- **副作用をテンプレートに書く**：補間内で API 呼び出しや代入をするのはアンチパターン。表示は純粋な式に保ち、ロジックは `<script setup>` 側へ。
- **巨大な三項のネスト**：読みづらくなったら `computed` に出す。

## 関連
[directives.md](./directives.md)
