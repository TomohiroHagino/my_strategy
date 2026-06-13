# Next.js 実務リファレンス（索引）

> **対象 = Next.js 15（App Router / React 19、TypeScript前提）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> Next.js は **React の上に乗るフルスタックフレームワーク**。ルーティング・SSR/SSG・データ取得・APIまで内蔵し、Reactだけでは足りない部分を埋める。

## この版のポイント（App Router 前提）
- **App Router（`app/`）が標準**：ファイル＝ルート。`page.tsx` / `layout.tsx` / `loading.tsx` / `error.tsx`。
- **React Server Components（RSC）が既定**：コンポーネントは**既定でサーバ実行**、対話が要る所だけ `"use client"`。
- **Server Actions**：フォーム送信やDB更新をサーバ関数で（`"use server"`）。
- **キャッシュモデル**：Next 15 で見直し（`fetch` は既定で都度取得寄りに、`revalidate` で制御）。
- 旧 **Pages Router**（`pages/`）はレガシー（既存案件で見る）。

## React との関係
```
 React        … UIを部品＋状態で組む「ライブラリ」（ブラウザ側）
 Next.js      … その上に ルーティング/SSR/データ取得/API を足した「フレームワーク」
```
→ Reactの基礎は [../react/](../../react/react19/) を前提に読む。

## 項目（各ファイルへ）

### はじめに / ルーティング
- [ハンズオン.md](./ハンズオン.md) … 【実習】0から作って動かす（create-next-app→page編集→Server Componentでfetch→ルート追加→"use client"を直す）
- [getting_started.md](./getting_started.md) … 始め方（create-next-app / app構成）
- [app_router.md](./app_router.md) … App Router（ファイルベースルーティング）とは
- [routing.md](./routing.md) … 動的ルート / Link / ナビゲーションとは
- [layouts.md](./layouts.md) … レイアウト / loading / error とは

### サーバ/クライアント・描画・データ
- [request_flow.md](./request_flow.md) … リクエストの流れ（middleware→page/route→RSC→HTML）・各部分が何を返すか
- [server_client_components.md](./server_client_components.md) … サーバ/クライアントコンポーネントとは（最重要）
- [data_fetching.md](./data_fetching.md) … データ取得（async サーバコンポーネント / fetch）とは
- [rendering.md](./rendering.md) … 描画戦略（SSR / SSG / ISR）とは
- [caching.md](./caching.md) … キャッシュ（Next 15 のモデル）とは

### 変更・API・運用
- [server_actions.md](./server_actions.md) … Server Actions（変更処理）とは
- [route_handlers.md](./route_handlers.md) … Route Handlers（API：route.ts）とは
- [middleware.md](./middleware.md) … ミドルウェアとは
- [edge_runtime.md](./edge_runtime.md) … Edge Runtime（エッジサーバー）とは
- [metadata_seo.md](./metadata_seo.md) … メタデータ / SEO とは
- [deployment.md](./deployment.md) … デプロイ（Vercel / 環境変数）とは

### テスト
- [testing_library.md](./testing_library.md) … React Testing Library（Client/Server Componentの注意点）とは
- [playwright.md](./playwright.md) … Playwright（App RouterページのE2E）とは

- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Next.js 15）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
