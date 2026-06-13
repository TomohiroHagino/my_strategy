# Nuxt 3 実務リファレンス（索引）

> **対象 = Nuxt 3（Vue 3 / Nitro / TypeScript前提）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> Nuxt は **Vue の上にルーティング・SSR・データ取得・サーバAPI を足したフレームワーク**。

## この版のポイント（Nuxt 3）
- **`pages/` のファイルベースルーティング**、`app.vue` / `layouts/`。
- **自動インポート**：`components/`・`composables/`・Vue/Nuxtの関数を import 不要で使える。
- **`useFetch` / `useAsyncData`**：SSRと相性の良いデータ取得（サーバで取ってハイドレーション）。
- **Nitro サーバ**：`server/api/` でAPIエンドポイントを同居。どこへでもデプロイ（presets）。
- **`useState`**：SSR安全な共有状態。

## Vue との関係
まず [../../vue/vue3/](../../vue/vue3/) の基礎（SFC・リアクティビティ・Composition API）を前提に読む。

## 項目（各ファイルへ）

### はじめに / ルーティング
- [ハンズオン.md](./ハンズオン.md) … 【実習】0から作って動かす（nuxi init→pages/でルーティング→useFetchでAPI表示→NuxtPage忘れを直す）
- [getting_started.md](./getting_started.md) … 始め方（nuxi init / ディレクトリ構成）
- [pages_routing.md](./pages_routing.md) … ページ / ファイルベースルーティングとは
- [components_autoimport.md](./components_autoimport.md) … 自動インポート（components / composables）とは
- [layouts_middleware.md](./layouts_middleware.md) … レイアウト / ルートミドルウェアとは

### データ・描画・サーバ
- [request_flow.md](./request_flow.md) … リクエストの流れ（server middleware→pages→useFetch→server/api→HTML）・各部分が何を返すか
- [data_fetching.md](./data_fetching.md) … データ取得（useFetch / useAsyncData）とは
- [rendering.md](./rendering.md) … 描画（SSR / SSG / SPA・ハイドレーション）とは
- [server_routes.md](./server_routes.md) … サーバAPI（Nitro / server/api）とは
- [state.md](./state.md) … 状態（useState / Pinia）とは
- [composables.md](./composables.md) … コンポーザブル（composables/）とは

### 設定・SEO・運用
- [config.md](./config.md) … 設定（nuxt.config / runtimeConfig / modules）とは
- [seo_meta.md](./seo_meta.md) … SEO / メタ（useHead / useSeoMeta）とは
- [deployment.md](./deployment.md) … デプロイ（Nitro presets）とは

### テスト
- [nuxt_test_utils.md](./nuxt_test_utils.md) … @nuxt/test-utils（mountSuspended / setup / Nuxt環境込み）とは
- [playwright.md](./playwright.md) … Playwright（実ブラウザE2E）とは

- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Nuxt 3）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
