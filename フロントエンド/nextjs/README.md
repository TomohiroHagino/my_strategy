# Next.js

## 一言で
**React の上に乗るフルスタックフレームワーク**。React だけでは足りないルーティング・SSR/SSG・データ取得・API を内蔵し、本番Webアプリを素早く作れる。現行は **App Router（React Server Components）** が標準。

## 特徴
- **App Router**：ファイル＝ルート（`app/`、`page.tsx`/`layout.tsx`）。
- **React Server Components 既定**：サーバ実行が基本、対話部分だけ `"use client"`。
- **Server Actions**：フォーム/DB更新をサーバ関数で。
- **描画戦略**：SSR / SSG / ISR を使い分け。SEOに強い。
- **Vercel と密**（デプロイが容易）。

## React との関係
```
 React    … UIを部品＋状態で組む「ライブラリ」
 Next.js  … その上に ルーティング/SSR/データ取得/API を足した「フレームワーク」
```
→ まず [../react/](../react/) の基礎、その上にこれ。

## 強み / 弱み
- 強み：フルスタック・SEO/性能・DX・エコシステム。
- 弱み：App Router/RSC/キャッシュの概念が難しめ（学習コスト）。

## このフォルダの構成
- [nextjs15/](./nextjs15/) … **Next.js 15 実務リファレンス（フラッグシップ）**。App Router〜Server/Client〜データ取得〜キャッシュ〜デプロイ〜罠まで、項目=1ファイル。
- [nextjs12/](./nextjs12/) … **Next.js 12 リファレンス（旧 Pages Router）**。App Router 以前の方式（`pages/`・`getServerSideProps` 等）。レガシー案件向け。
