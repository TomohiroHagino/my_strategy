# 始め方（Vue 3）

## ひとことで言うと
Vue 3 プロジェクトを **Vite ベースの公式ツール `create-vue`** で立ち上げ、SFC（`.vue`）を書き、`createApp(App).mount('#app')` で DOM に載せ、`npm run dev` で動かすまでの一連の流れ。

## 役割・なぜ必要か
- Vue は「**状態(ref)を変えると、それを使うテンプレートが自動で再描画される**」フレームワーク。その最小単位が SFC（Single File Component）で、1ファイルに `<template>` / `<script setup>` / `<style>` をまとめて書ける。
- 素の `<script>` 読み込みでも動くが、実務では **ビルドツール（Vite）+ SFC + TypeScript** が標準。`.vue` のコンパイル、型チェック、HMR（編集即反映）がまとめて手に入る。
- `create-vue` は公式の足場ツール。TypeScript / Router / Pinia / Vitest などを **対話的に選択**してテンプレートを生成するので、自前で webpack 等を組む必要がない。

## 基本の書き方（コード）
プロジェクト作成（対話形式でオプションを選ぶ）:
```bash
# Node 18+ 推奨。npm create は create-vue を呼び出す
npm create vue@latest

# 例: 質問に答える（実務の定番）
# Project name: my-app
# Add TypeScript?            → Yes
# Add JSX?                   → No
# Add Vue Router?            → Yes（SPAでページ遷移するなら）
# Add Pinia?                 → Yes（状態管理するなら）
# Add Vitest?                → Yes（単体テスト）
# Add ESLint / Prettier?     → Yes

cd my-app
npm install
npm run dev          # http://localhost:5173 が立ち上がる
```

エントリポイント `src/main.ts`:
```ts
import { createApp } from 'vue'
import App from './App.vue'   // ルートとなる SFC
import './assets/main.css'

// App コンポーネントを #app（index.html 内の <div id="app">）にマウント
createApp(App).mount('#app')
```

ルート SFC `src/App.vue`（`<script setup lang="ts">` 形式）:
```vue
<script setup lang="ts">
import { ref } from 'vue'

// ref = リアクティブな状態。値は .value で読み書きする
const count = ref(0)
const increment = () => { count.value++ }
</script>

<template>
  <main>
    <h1>Hello Vue 3</h1>
    <!-- 補間とイベント。count が変わると自動で再描画される -->
    <button @click="increment">count is {{ count }}</button>
  </main>
</template>

<style scoped>
h1 { color: #42b883; }
</style>
```

## 実務での使い方・定番パターン
- **TypeScript はほぼ必須**：`<script setup lang="ts">` を選び、props/emit に型を付ける。型補完と安全性が段違い。→ [components.md](./components.md)
- **`<script setup>` を標準に**：`setup()` を手書きする旧来形より圧倒的に短く書ける。import した変数・関数はそのままテンプレートで使える。→ [reactivity.md](./reactivity.md)
- **ディレクトリ構成**：`src/components/`（部品）、`src/views` や `src/pages`（Router 連携）、`src/stores/`（Pinia）、`src/composables/`（再利用ロジック）に分ける。
- **`npm run build` で本番ビルド**（`dist/` 生成）、`npm run preview` で本番相当を確認。`vite.config.ts` で alias（`@` → `src`）やプラグインを設定。
- 既存ページに**部分的に組み込む**場合も `createApp(...).mount('#特定要素')` で局所マウントできる。

## ハマりどころ / アンチパターン
- **`create-vue` のオプション選択ミス**：後から Router/Pinia を足すことは可能だが、最初に必要なものを選んでおくと設定が自動で入って楽。Vue CLI（`@vue/cli`）は旧式なので**新規は `create-vue`（Vite）を使う**。
- **`ref` の `.value` 付け忘れ**：`<script>` 内では `count.value`、テンプレート内では `count`（自動アンラップ）。この差で混乱しやすい。→ [reactivity.md](./reactivity.md)
- **SFC の構造順序**：`<script setup>` → `<template>` → `<style>` の順が慣例だが順序自体は自由。ただし `<style scoped>` を付け忘れると CSS がグローバルに漏れる。
- **`mount('#app')` の対象が無い**：`index.html` に `<div id="app"></div>` が無い／IDが違うと何も表示されない。
- **Node のバージョン不足**：古い Node だと `npm create vue@latest` が失敗する。LTS（18+ 推奨）を使う。
- **`main.ts` で App をマウントし忘れ**：`createApp(App)` だけで `.mount()` を呼ばないと画面に出ない。

## フォルダ構成（始動直後）
```
myapp/
├── index.html              # エントリHTML（<div id="app"> を持つ）
├── src/
│   ├── main.ts             # createApp(App).mount('#app')
│   ├── App.vue             # ルートとなる SFC
│   ├── components/         # 部品（雛形に数点含まれる）
│   ├── assets/             # CSS・画像など
│   ├── router/             # Vue Router 選択時に生成
│   ├── stores/             # Pinia 選択時に生成
│   └── views/              # Router 選択時のページ用 SFC
├── public/
│   └── favicon.ico         # そのまま配信される静的ファイル
├── vite.config.ts          # Vite の設定（@ → src の alias 等）
├── tsconfig.json           # TS設定（TS 選択時）
├── env.d.ts                # 型定義（JS構成では jsconfig.json）
├── package.json            # scripts: dev / build / preview
├── .gitignore              # node_modules 等を除外
└── node_modules/           # 依存パッケージ（npm install で生成）
```
- 作成時のプロンプトで Router / Pinia / TS を選ぶと対応フォルダ（router/ stores/ views/）が生成される。

## 関連
[components.md](./components.md) / [reactivity.md](./reactivity.md)
