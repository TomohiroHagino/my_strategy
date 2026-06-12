# Vue 3 実務リファレンス（索引）

> **対象 = Vue 3（Composition API / `<script setup>` / TypeScript前提）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> Vue の核は「**テンプレート × リアクティビティ**：状態(ref)を変えると、それを使うテンプレートが自動で再描画される」。

## この版のポイント（Vue 3）
- **Composition API（`<script setup>`）が標準**：ロジックを関数的に組む（旧 Options API も使える）。
- **リアクティビティの刷新**：`ref` / `reactive`（Proxyベース）。
- **複数ルート要素OK**（Fragments）、Teleport、Suspense など。
- Vue 2 からの移行（`Vue.createApp`、フィルタ廃止 等）。

## 項目（各ファイルへ）

### はじめに / 基礎
- [getting_started.md](./getting_started.md) … 始め方（create-vue / SFC / mount）
- [template_syntax.md](./template_syntax.md) … テンプレート構文（補間 / `:` / `@`）とは
- [components.md](./components.md) … コンポーネント（SFC・親子）とは
- [directives.md](./directives.md) … ディレクティブ（v-if / v-for / v-model）とは
- [slots.md](./slots.md) … スロット（コンテンツ差し込み）とは

### リアクティビティ / ロジック
- [reactivity.md](./reactivity.md) … リアクティビティ（`ref` / `reactive`）とは
- [computed_watch.md](./computed_watch.md) … `computed` / `watch` とは
- [composition_api.md](./composition_api.md) … Composition API（`<script setup>`）とは
- [props_emit.md](./props_emit.md) … props / emit / コンポーネントの v-model とは
- [lifecycle.md](./lifecycle.md) … ライフサイクル（onMounted 等）とは

### 周辺・運用
- [request_flow.md](./request_flow.md) … データの流れ（状態→UIの一方向＋データ取得フロー）・各部分が何を返すか
- [pinia.md](./pinia.md) … 状態管理（Pinia）とは
- [routing.md](./routing.md) … ルーティング（Vue Router）とは
- [testing.md](./testing.md) … テスト（Vitest / Vue Test Utils）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

> 関連: フルスタック版は [../../nuxt/nuxt3/](../../nuxt/nuxt3/)。

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Vue 3）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
