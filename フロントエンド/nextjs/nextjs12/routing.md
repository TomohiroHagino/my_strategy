# ルーティング（ファイルベース / 動的 / Link / useRouter）（Next.js 12・Pages Router）

## ひとことで言うと
`pages/` 配下のファイルパスがそのままURLになり、`[id].tsx` のような角括弧で動的セグメントを受け取る——画面間の移動は `next/link` の `<Link>`、現在地やクエリは `next/router` の `useRouter()` で扱う。

## 役割・なぜ必要か
- ルーティング表をどこかに書く必要がなく、**ファイルを置くだけでURLが増える**。学習コストが低く、構成を見ればURLが分かる。
- ブログ記事や商品ページのように件数が可変なページは、1ファイル `[id].tsx` で全URLを賄える。`router.query` または `getStaticProps`/`getServerSideProps` の `params` でその値を受け取る。
- 内部リンクを `<a>` ではなく `<Link>` にすると、Next がプリフェッチ＆クライアント遷移し、**全ページ再読み込みを避ける**（速い・状態が保てる）。

## 基本の書き方（コード）

ファイル＝URLの対応（`pages/` 配下）。
```
pages/index.tsx            → "/"
pages/about.tsx            → "/about"
pages/blog/index.tsx       → "/blog"
pages/blog/[slug].tsx      → "/blog/xxx"（動的・1セグメント）
pages/docs/[...slug].tsx   → "/docs/a/b/c"（catch-all・配列）
pages/shop/[[...slug]].tsx → "/shop" も "/shop/a/b" も（任意catch-all）
pages/api/hello.ts         → "/api/hello"（APIルート → api_routes.md）
```

```tsx
// pages/index.tsx → "/"
export default function Home() {
  return <h1>こんにちは Pages Router</h1>;
}
```

```tsx
// pages/blog/[slug].tsx → "/blog/xxx"
// クライアント側で query から取る場合（データ取得関数を使わないとき）
import { useRouter } from "next/router";

export default function Post() {
  const router = useRouter();
  const { slug } = router.query; // 例: /blog/hello → slug = "hello"（string | string[] | undefined）
  return <h1>記事: {slug}</h1>;
}
```

```tsx
// catch-all: [...slug].tsx は複数セグメントを配列で受ける
// pages/docs/[...slug].tsx → /docs/a/b/c で query.slug = ["a","b","c"]
import { useRouter } from "next/router";

export default function Docs() {
  const { slug } = useRouter().query; // string[] | undefined
  return <p>{Array.isArray(slug) ? slug.join(" / ") : ""}</p>;
}
```

```tsx
// <Link> でのナビゲーション（内部リンクは <a> ではなくこれ）
import Link from "next/link";

export default function Nav() {
  return (
    <nav>
      <Link href="/">Home</Link>
      <Link href="/blog/hello">記事へ</Link>
      <Link href={{ pathname: "/search", query: { q: "next" } }}>検索</Link>
    </nav>
  );
}
```

```tsx
// useRouter でプログラム遷移・現在地・クエリ
import { useRouter } from "next/router";

export function Controls() {
  const router = useRouter();
  // router.pathname … "/blog/[slug]"（ルート定義）
  // router.asPath   … "/blog/hello?x=1"（実URL）
  // router.query    … { slug: "hello", x: "1" }
  return (
    <button onClick={() => router.push("/")}>戻る（now: {router.asPath}）</button>
  );
}
```

> Next.js 12 系の `next/link` は子に `<a>` を要求するバージョン（`<Link href="/"><a>Home</a></Link>`）と、`<a>` 不要のバージョンが混在する。`next@^12.2` 以降は `<Link href="/">Home</Link>` で動くが、古い 12.0 系プロジェクトでは `<a>` を内側に置く点に注意（→ `pitfalls.md`）。

## 実務での使い方・定番パターン
- **一覧→詳細**：一覧で `<Link href={`/blog/${post.slug}`}>` を並べ、詳細 `[slug].tsx` で `getStaticProps`/`getServerSideProps` の `params.slug` からデータを引く（→ [data_fetching.md](./data_fetching.md)）。
- **クエリでフィルタ/ページング/タブ**：`router.push({ pathname, query })` でURLにクエリを乗せ、`router.query` から読む。URL がそのまま共有・ブックマーク可能になる。
- **動的値はデータ取得関数の `params` 優先**：SSG/SSR なら `useRouter().query` よりも `getStaticProps`/`getServerSideProps` の `context.params` で受けるとサーバ側で確定する（`fallback` 中の `undefined` を踏まない）。
- **遷移完了の検知**：`router.events`（`routeChangeStart`/`routeChangeComplete`）でローディングバーやアナリティクス送信を仕込む。

## ハマりどころ / アンチパターン
- **`<a href>` で内部遷移**：全ページ再読み込みになりプリフェッチも効かない。内部リンクは必ず `<Link>`。外部URLだけ `<a>`。
- **`next/navigation` から import**：それは App Router 用。Pages Router は **`next/router` の `useRouter`** が正（`usePathname`/`useSearchParams` はApp Router）。App Router では `next/navigation`（[../nextjs15/](../nextjs15/)）。
- **`router.query` が初回レンダーで空**：静的最適化されたページや `fallback` 中は `query` が空オブジェクトになり得る。`router.isReady` を見るか、データ取得関数の `params` を使う。
- **`query` の値の型**：`string | string[] | undefined`。動的1セグメントでも catch-all の都合で配列になる場面があり、`Array.isArray` チェックを怠ると落ちる。
- **動的ルートのファイル名衝突**：同階層に `[id].tsx` と `[slug].tsx` を両方置くと曖昧で衝突する。動的セグメントは1階層1つ。

## 関連
[data_fetching.md](./data_fetching.md) / [rendering.md](./rendering.md) / [api_routes.md](./api_routes.md) / [pitfalls.md](./pitfalls.md)
