# _app.tsx と _document.tsx（共通枠 / html構造）（Next.js 12・Pages Router）

## ひとことで言うと
`_app.tsx` は全ページを包む共通ラッパー（global CSS の import・Provider・レイアウト）で、`_document.tsx` は出力HTMLの `<html>`/`<body>` の骨組みをサーバ側だけでカスタムする場所——役割が違う2つの特別ファイル。

## 役割・なぜ必要か
- **`_app.tsx`**：全ページに共通して必要なもの（global CSS、テーマ/認証/状態管理の Provider、共通レイアウト）を1か所で巻く。App Router の `layout.tsx` に相当する役割を担う。
- **`_document.tsx`**：`<html lang>`・`<body>` の属性、フォントの `<link>`、`lang` 指定など**ドキュメント全体の骨組み**を変えたいとき。**サーバでのみ実行**され、イベントハンドラや `useEffect` は書けない（描画ロジックではなく構造の定義）。
- ページ単位の `<title>`/`<meta>` は `_document` ではなく **`next/head` の `<Head>`** で行う（→ [metadata_seo.md](./metadata_seo.md)）。

## 基本の書き方（コード）

```tsx
// pages/_app.tsx … 全ページ共通ラッパー
import type { AppProps } from "next/app";
import "../styles/globals.css"; // ← global CSS の import はここでだけ可能

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}
```

```tsx
// pages/_app.tsx … Provider を巻く例（テーマ / React Query など）
import type { AppProps } from "next/app";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import "../styles/globals.css";

const queryClient = new QueryClient();

export default function App({ Component, pageProps }: AppProps) {
  return (
    <QueryClientProvider client={queryClient}>
      <Component {...pageProps} />
    </QueryClientProvider>
  );
}
```

```tsx
// pages/_app.tsx … getLayout パターン（ページごとに違うレイアウトを差す）
import type { ReactElement, ReactNode } from "react";
import type { NextPage } from "next";
import type { AppProps } from "next/app";

type NextPageWithLayout = NextPage & {
  getLayout?: (page: ReactElement) => ReactNode;
};
type AppPropsWithLayout = AppProps & { Component: NextPageWithLayout };

export default function App({ Component, pageProps }: AppPropsWithLayout) {
  const getLayout = Component.getLayout ?? ((page) => page);
  return getLayout(<Component {...pageProps} />);
}

// 各ページ側：
// Page.getLayout = (page) => <DashboardLayout>{page}</DashboardLayout>;
```

```tsx
// pages/_document.tsx … HTMLの骨組み（任意・カスタムする時だけ作る・サーバのみ実行）
import { Html, Head, Main, NextScript } from "next/document";

export default function Document() {
  return (
    <Html lang="ja">
      <Head>
        {/* ここは全ページ共通の <link>/<meta>。ページ個別の title は next/head で */}
        <link rel="preconnect" href="https://fonts.googleapis.com" />
      </Head>
      <body>
        <Main />        {/* ← ページ本体が入る。消すと何も描画されない */}
        <NextScript />  {/* ← Next のスクリプト。消すと動かない */}
      </body>
    </Html>
  );
}
```

## 実務での使い方・定番パターン
- **global CSS は `_app.tsx` でだけ import**：他の場所での global CSS import はビルドエラーになる（CSS Modules は各コンポーネントで可）。
- **Provider はまとめて `_app.tsx`**：認証・テーマ・i18n・状態管理など、全ページ共通のコンテキストはここで一括して巻く。
- **共通ヘッダ/フッタ**：単純なら `_app.tsx` で `<Layout>` を巻き、ページごとに変えたいなら `getLayout` パターン。
- **`_document.tsx` は最小限**：`lang` 属性やフォントの preconnect 程度。基本は作らなくてよい（無くても動く）。
- **`next/document` の `<Head>` ≠ `next/head` の `<Head>`**：前者はドキュメント全体（全ページ共通の静的要素）、後者はページ単位の動的メタ。混同しない。

## ハマりどころ / アンチパターン
- **App Router の `layout.tsx` を持ち込む**：Pages Router に `layout.tsx` の規約はない。共通枠は `_app.tsx`／`getLayout`（[../nextjs15/](../nextjs15/)）。
- **`_document.tsx` で `<Main />`/`<NextScript />` を消す/動かす**：`<Html><Head /><body><Main /><NextScript /></body></Html>` の構造を崩すと描画やスクリプトが壊れる。
- **`_document.tsx` にイベント/フック**：サーバのみ実行なので `onClick`/`useEffect`/`useState` は無効。対話やブラウザ依存は `_app` 以降のコンポーネントへ。
- **`_document` でページ個別の `<title>`**：`_document` は全ページ共通。ページごとの title/meta は `next/head` の `<Head>` で（→ [metadata_seo.md](./metadata_seo.md)）。
- **global CSS をコンポーネントで import**：`_app.tsx` 以外での global CSS import はエラー。スコープ付きが要るなら CSS Modules（→ [styling.md](./styling.md)）。

## 関連
[styling.md](./styling.md) / [metadata_seo.md](./metadata_seo.md) / [routing.md](./routing.md) / [pitfalls.md](./pitfalls.md)
