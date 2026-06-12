# コンポーネント（Vue 3）

## ひとことで言うと
UI を再利用可能な部品に分けた単位。Vue では **SFC（Single File Component / `.vue`）** に `<template>`（見た目）・`<script setup>`（ロジック）・`<style scoped>`（その部品専用CSS）をまとめて書く。

## 役割・なぜ必要か
- 画面を**部品の組み合わせ**で作ることで、再利用・テスト・保守がしやすくなる（ボタン、カード、フォーム…を1ファイルずつ）。
- 親が子を **import して使い**、データは **親→子へ props**、通知は **子→親へ emit** で流す。この一方向の流れ（単方向データフロー）が状態の追跡を容易にする。
- `<style scoped>` により CSS がその部品内だけに閉じ、**他コンポーネントへの style 漏れ**を防げる。

## 基本の書き方（コード）
子コンポーネント `src/components/UserCard.vue`:
```vue
<script setup lang="ts">
// 親から受け取る props を型で定義
const props = defineProps<{ name: string; age: number }>()

// 親へ通知するイベントを定義
const emit = defineEmits<{ (e: 'select', id: string): void }>()

const onClick = () => emit('select', props.name)
</script>

<template>
  <div class="card" @click="onClick">
    <h3>{{ name }}</h3>     <!-- props は直接参照できる -->
    <p>{{ age }}歳</p>
  </div>
</template>

<style scoped>
/* この .card は UserCard 内だけに適用される */
.card { padding: 12px; border: 1px solid #ddd; cursor: pointer; }
</style>
```

親コンポーネント `src/App.vue`:
```vue
<script setup lang="ts">
import { ref } from 'vue'
// import するだけで使える（明示的な components 登録は不要）
import UserCard from './components/UserCard.vue'

const selected = ref('')
const handleSelect = (name: string) => { selected.value = name }
</script>

<template>
  <!-- props は属性で渡す。数値などは : でバインドして式として渡す -->
  <UserCard name="Taro" :age="20" @select="handleSelect" />
  <p>選択中: {{ selected }}</p>
</template>
```

## 実務での使い方・定番パターン
- **`<script setup>` では子のimport＝自動登録**：`import Child from '...'` すればテンプレートでそのまま `<Child />` と書ける（旧 Options API の `components: { Child }` は不要）。
- **props は読み取り専用**：子は受け取った props を変更しない。変更が必要なら emit で親に依頼し、親が状態を更新する（単方向データフロー）。詳細 → [props_emit.md](./props_emit.md)
- **命名**：SFC は `PascalCase`（`UserCard.vue`）。テンプレート内は `<UserCard />`（PascalCase）でも `<user-card />`（kebab-case）でも使えるが、**プロジェクト内で統一**する。
- **`<style scoped>`** を基本にし、グローバルに当てたいものだけ別途グローバルCSSへ。深い子へ当てたいときは `:deep(.selector)` を使う。
- 再利用ロジックは composable（`src/composables/useXxx.ts`）に切り出し、表示は SFC に集約する。
- 子へ任意のテンプレートを差し込むなら slot を使う。→ [slots.md](./slots.md)

## ハマりどころ / アンチパターン
- **scoped CSS の範囲を誤解**：`<style scoped>` は**そのコンポーネントの要素にだけ**当たる。子コンポーネント内部の要素には基本届かない（`:deep()` が必要）。逆に scoped を付け忘れると全画面に漏れる。
- **PascalCase / kebab-case の混在**：`<UserCard>` と `<user-card>` を混ぜると統一感が崩れ grep もしづらい。どちらかに決める。
- **props を子で書き換える**：`props.age = 30` は警告 & アンチパターン。親が持つ状態を子が直接いじると追跡不能になる。emit 経由で親に変更させる。→ [props_emit.md](./props_emit.md)
- **props の渡し方ミス**：`:age="20"`（数値）と `age="20"`（文字列 "20"）は別物。式・数値・真偽値は `:` でバインドする。
- **巨大SFC化**：1ファイルが膨らんだら子コンポーネントや composable に分割する（高凝集・低結合）。
- **import 忘れ**：`<script setup>` で import していない部品をテンプレートで使うと未定義になる。

## 関連
[props_emit.md](./props_emit.md) / [slots.md](./slots.md)
