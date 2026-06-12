# ディレクティブ（Directives）（Vue 3）

## ひとことで言うと
テンプレートのタグに付ける `v-` で始まる**特別な属性**で、「状態に応じてDOMをどう描画・操作するか」を宣言的に指定する仕組み。`v-if`・`v-for`・`v-model` などが代表。

## 役割・なぜ必要か
- JSで手作業に `createElement` や `addEventListener` を書く代わりに、**HTML側に「条件・繰り返し・バインド」を宣言**して、状態が変われば自動でDOMへ反映させる。
- Vueの核「**状態 → テンプレート自動再描画**」を実現する接着剤。状態(`ref`)を変えるだけで、ディレクティブが描画結果を追従させる。→ [reactivity.md](./reactivity.md)
- よく使うのは `v-bind`(`:`属性)、`v-on`(`@`イベント)、`v-if`/`v-show`(表示制御)、`v-for`(繰り返し)、`v-model`(双方向)。

## 基本の書き方（コード）
```vue
<script setup lang="ts">
import { ref } from 'vue'

const isOpen = ref(false)
const keyword = ref('')
const items = ref([
  { id: 1, name: 'りんご' },
  { id: 2, name: 'みかん' },
])
</script>

<template>
  <!-- v-bind：属性に状態を結ぶ（: が省略記法） -->
  <a :href="`/items?q=${keyword}`">検索</a>

  <!-- v-on：イベント購読（@ が省略記法） -->
  <button @click="isOpen = !isOpen">トグル</button>

  <!-- v-if / v-else-if / v-else：条件でDOMを「付け外し」する -->
  <p v-if="items.length === 0">データなし</p>
  <ul v-else>
    <!-- v-for：:key 必須。id など安定した一意キーを使う -->
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  </ul>

  <!-- v-show：条件で display を切り替える（DOMは常に存在） -->
  <div v-show="isOpen">開閉する詳細パネル</div>

  <!-- v-model：入力と状態の双方向バインド -->
  <input v-model="keyword" placeholder="キーワード" />
  <p>入力中: {{ keyword }}</p>
</template>
```

## 実務での使い方・定番パターン
- **`v-if` vs `v-show` の使い分け**：
  - `v-if` … 条件が `false` の間は**DOM自体を生成しない**（付け外し）。初期コストは軽いが、切替コストは高め。**滅多に切り替わらない**ものや、重い子コンポーネントに向く。
  - `v-show` … 常にレンダリングし、`display: none` を**CSSで切り替える**だけ。**頻繁にトグルする**UI（タブ・アコーディオン）に向く。
- **`v-for` の `:key`**：差分検知の目印。**安定した一意ID（`item.id`）を必ず付ける**。並べ替え・追加・削除でも要素の同一性が保たれ、再描画と状態保持が正しくなる。
- **`v-bind` 省略 `:` / `v-on` 省略 `@`**：実務ではほぼ省略記法を使う。`:class`・`:style` はオブジェクト/配列構文でクラス切替が書ける（`:class="{ active: isOpen }"`）。
- **`v-model` の修飾子**：`.trim`（前後空白除去）、`.number`（数値化）、`.lazy`（change時に同期）。フォーム入力で多用。
- **カスタムディレクティブ**：DOM直接操作（フォーカス・外部ライブラリ連携）が必要な時だけ `v-focus` のように自作する。多用は避ける。

## ハマりどころ / アンチパターン
- **`:key` に配列の `index` を使う**（最頻出アンチパターン）：途中で要素を追加・削除・並べ替えすると、indexがズレて**別要素に状態が紐づく**（入力値や `v-show` 状態が混線、チェックボックスがズレる）。**必ず `item.id` など安定キー**を使う。
- **`v-if` と `v-for` を同一要素で併用しない**：Vue 3 では `v-if` が `v-for` より優先され、`item` がまだ未定義で参照エラーになる。**外側に `<template v-for>` を置き、内側で `v-if`** に分ける（または `computed` で事前フィルタ）。→ [reactivity.md](./reactivity.md)
  ```vue
  <!-- NG: 同一要素で併用 -->
  <li v-for="u in users" v-if="u.active" :key="u.id">{{ u.name }}</li>
  <!-- OK: template で v-for、内側で v-if -->
  <template v-for="u in users" :key="u.id">
    <li v-if="u.active">{{ u.name }}</li>
  </template>
  ```
- **頻繁にトグルするのに `v-if`**：切替のたびにDOMを作り直して無駄。トグルUIは `v-show` を使う。
- **`@click="handler()"` と `@click="handler"` の混同**：引数なしで渡すなら `@click="handler"`（イベント引数が自動で渡る）。`()` を付けると即時実行になり意図とズレる。
- **`v-html` での生HTML挿入**：XSSの温床。ユーザー入力を `v-html` に入れない。

## 関連
[components.md](./components.md) / [reactivity.md](./reactivity.md) / [slots.md](./slots.md)
