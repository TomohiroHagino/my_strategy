# メタデータ / SEO（next/head・next/script）（Next.js 12・Pages Router）

## ひとことで言うと
ページ単位の `<title>`/`<meta>`/OGP は **`next/head` の `<Head>`** で `<head>` に注入し、外部スクリプトは **`next/script`** で読み込みタイミングを制御する——Pages Router でのSEO・メタ管理の基本。

## 役割・なぜ必要か
- 検索結果やSNSシェアの見た目（タイトル・説明・OGP画像）は `<head>` の中身で決まる。`<Head>` を使うと**ページごとに違うメタ**を宣言でき、サーバ生成HTMLに含まれるのでクローラやSNSが読める。
- スクリプトを素の `<script>` で置くとレンダリングを止めたり順序が乱れる。`next/script` の `strategy` で**いつ読むか**を制御し、パフォーマンスを守る。
- `_document.tsx` は全ページ共通の骨組み専用。**ページ個別**のメタはここではなく `next/head`（→ [app_document.md](./app_document.md)）。

## 基本の書き方（コード）

```tsx
// ページ単位の title / meta / OGP
import Head from "next/head";

export default function Article() {
  return (
    <>
      <Head>
        <title>記事タイトル | サイト名</title>
        <meta name="description" content="この記事の説明文（120字程度）" />
        {/* OGP（SNSシェア時の見た目） */}
        <meta property="og:title" content="記事タイトル" />
        <meta property="og:description" content="この記事の説明文" />
        <meta property="og:image" content="https://example.com/og.png" />
        <meta property="og:type" content="article" />
        {/* Twitter Card */}
        <meta name="twitter:card" content="summary_large_image" />
        {/* 正規URL（重複URL対策） */}
        <link rel="canonical" href="https://example.com/articles/1" />
      </Head>
      <article>本文…</article>
    </>
  );
}
```

```tsx
// 共通のデフォルトメタは _app.tsx の <Head> に置き、各ページで上書きできる
import Head from "next/head";
import type { AppProps } from "next/app";

export default function App({ Component, pageProps }: AppProps) {
  return (
    <>
      <Head>
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <title>サイト名（デフォルト）</title>
      </Head>
      <Component {...pageProps} />
    </>
  );
}
```

```tsx
// next/script で外部スクリプトの読み込みタイミングを制御
import Script from "next/script";

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <>
      {/* afterInteractive（既定）：ページが対話可能になった後。アナリティクス等 */}
      <Script src="https://example.com/analytics.js" strategy="afterInteractive" />
      {/* lazyOnload：アイドル時。チャットウィジェット等の優先度低いもの */}
      <Script src="https://example.com/chat.js" strategy="lazyOnload" />
      {/* beforeInteractive：ハイドレーション前に必要なもの（限定的に） */}
      {children}
    </>
  );
}
```

JSON-LD（構造化データ）。
```tsx
<Head>
  <script
    type="application/ld+json"
    dangerouslySetInnerHTML={{
      __html: JSON.stringify({
        "@context": "https://schema.org",
        "@type": "Article",
        headline: "記事タイトル",
      }),
    }}
  />
</Head>
```

## 実務での使い方・定番パターン
- **デフォルト＋上書き**：`_app.tsx` の `<Head>` に共通の `viewport`/既定 title を置き、各ページの `<Head>` で title/description を上書き（同名の `<title>` は後勝ち）。
- **動的メタ**：`getStaticProps`/`getServerSideProps` で取った値を `<Head>` に流し込み、記事ごとに違うOGPを出す。
- **`next/image`**：画像最適化（サイズ・遅延読み込み・WebP変換）でCWVを改善。`width`/`height` を明示してCLSを防ぐ。
- **`strategy` の選択**：計測系は `afterInteractive`、優先度低いウィジェットは `lazyOnload`、ポリフィル等の必須は `beforeInteractive`。
- **`sitemap.xml`/`robots.txt`**：`public/` に静的配置するか、`pages/sitemap.xml.tsx` 等で `getServerSideProps` から動的生成。

## ハマりどころ / アンチパターン
- **App Router の `metadata` API を持ち込む**：`export const metadata`／`generateMetadata` は App Router 専用（[../nextjs15/](../nextjs15/)）。Pages Router は `next/head` の `<Head>`。
- **`_document.tsx` にページ個別の title**：`_document` は全ページ共通。個別メタは `next/head`（→ [app_document.md](./app_document.md)）。
- **`<title>` の重複/未設定**：複数の `<Head>` で同名タグは後勝ち。各ページで title を必ず設定しないと一覧で同じ title になる。
- **生の `<script>` を本文に直書き**：レンダリングブロックや二重読み込みの原因。`next/script` を使い `strategy` を指定。
- **CSRページでメタが空**：`useEffect` 取得だけの画面は初期HTMLに本文が無くSEO/OGPが弱い。重要ページはサーバ生成（→ [rendering.md](./rendering.md)）。
- **OGP画像が絶対URLでない**：`og:image` は相対パスだとSNSが解決できない。完全なURLを入れる。

## 関連
[app_document.md](./app_document.md) / [rendering.md](./rendering.md) / [data_fetching.md](./data_fetching.md) / [pitfalls.md](./pitfalls.md)
