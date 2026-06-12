# 自動インポート（components / composables）（Nuxt 3）

## ひとことで言うと
**`components/` 配下のコンポーネントや `composables/` の関数、さらに Vue/Nuxt 標準の関数（`ref`・`useFetch` 等）を、`import` を書かずにそのまま使える**仕組み。Nuxt がビルド時に必要な import を自動で挿入する。

## 役割・なぜ必要か
- 毎回 `import MyButton from '~/components/MyButton.vue'` を書くのは定型作業。**配置＝利用可能** にして記述量とミスを減らす。
- どこからでも同じ名前で呼べるので、**コンポーネント名・関数名が事実上の共通語彙**になる（チームで揃う）。
- import 経路の打ち間違い・相対パス地獄が消える。リファクタでファイルを移動しても呼び出し側を直さなくてよい。

## 基本の書き方（コード）
```bash
components/
├─ AppButton.vue            # → <AppButton />
└─ base/
    └─ Card.vue             # → <BaseCard />  ※フォルダ名がプレフィックス
composables/
├─ useCounter.ts            # → useCounter() がそのまま使える
└─ useAuth.ts               # → useAuth()
```

```vue
<!-- components/AppButton.vue -->
<template>
  <button class="app-btn"><slot /></button>
</template>
```

```ts
// composables/useCounter.ts
export function useCounter() {       // 関数名 = 呼び出し名
  const count = ref(0)               // ref も自動インポート（import 不要）
  const inc = () => count.value++
  return { count, inc }
}
```

```vue
<!-- pages/index.vue：import 文がひとつも無い -->
<script setup lang="ts">
// ref / useCounter / useFetch すべて import なしで使える
const { count, inc } = useCounter()
const { data } = await useFetch('/api/hello')
</script>

<template>
  <AppButton @click="inc">count: {{ count }}</AppButton>
  <BaseCard>{{ data }}</BaseCard>     <!-- base/Card.vue が BaseCard -->
</template>
```

## 実務での使い方・定番パターン
- **コンポーネント名は「フォルダ + ファイル名」のパスカルケース**。`base/Card.vue` → `<BaseCard />`、`form/text/Field.vue` → `<FormTextField />`。命名衝突を防ぐ。
- **`composables/` のトップレベルの `export`** が自動インポート対象。`export function useX` / `export const useX` どちらでも可。慣例として `use` 接頭辞を付ける。→ [composables.md](./composables.md)
- **明示 import との混在も可**。外部パッケージや `~/utils/` の純粋関数は普通に `import` する。自動化はあくまで規約フォルダ内。
- **遅延読み込み**は `<LazyHeavyChart />` のように `Lazy` プレフィックスで動的インポートになる（初期バンドルを軽く）。
- **設定で広げられる**：`nuxt.config.ts` の `components: [{ path: '~/widgets', prefix: 'W' }]` で対象ディレクトリやプレフィックスを追加。→ [config.md](./config.md)

```ts
// nuxt.config.ts：自動インポート対象を足す例
export default defineNuxtConfig({
  components: [
    { path: '~/components', pathPrefix: false }, // フォルダ名をプレフィックスにしない
  ],
})
```

## ハマりどころ / アンチパターン
- **命名規則の誤解**で「コンポーネントが見つからない」。`base/Card.vue` は `<Card />` ではなく **`<BaseCard />`**。パスがそのまま名前に乗る。
- **同名 `import` との二重定義**：自分で `import` した名前と自動インポート名が衝突すると曖昧になる。どちらか一方に統一する。
- **`composables/` のサブフォルダはトップレベル export のみ自動対象**。深い階層やデフォルトエクスポートは拾われないことがある。`use*` を名前付き export する。
- **IDE が `<AppButton>` や `useCounter` を「未定義」と赤線**にする時は、**`.nuxt/` の型が古い**のが原因。`npm run dev` を一度起動するか `npx nuxi prepare`（型再生成）で直る。
- **`utils/` を自動インポート前提**にしてしまう（Nuxt 3 既定では `composables/` と `components/` 中心）。共有関数は `composables/` に置くか、明示 import する。
- **大量の `Lazy` 乱用**で逆に遅くなる。本当に重い・初期に不要な物だけ遅延化する。

## 関連
[composables.md](./composables.md) / [config.md](./config.md) / [getting_started.md](./getting_started.md)
