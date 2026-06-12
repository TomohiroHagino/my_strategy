# はじめに（getting started）（Next.js 12・Pages Router）

## ひとことで言うと
**App Router 登場前の、Next.js の従来型ルーティング**。`pages/` にファイルを置くと、そのパスがそのままURLになる。データ取得は `getServerSideProps` / `getStaticProps` という専用関数で行う。Next.js 12 までの既定方式で、13〜15 でも併存・サポートされる。

## 役割・なぜ必要か
- App Router 以前のNext.js標準。**既存の多くの案件・解説記事はこの方式**なので、レガシー保守で必ず出会う。
- `pages/` にファイルを置くだけでルートになる、学習コストの低い直感的モデル。
- App Router（Server Components）と違い、**「全部クライアント側のReact＋専用データ取得関数」**という分かりやすい構造。

## App Router（nextjs15）との違い（要点）
| | Pages Router（旧・本書） | App Router（新・[nextjs15](../nextjs15/)） |
|---|---|---|
| ルート定義 | `pages/` のファイル＝ルート | `app/` のフォルダ＝ルート |
| 画面ファイル | `pages/about.tsx` | `app/about/page.tsx` |
| 共通の枠 | `_app.tsx` / `_document.tsx` | `layout.tsx` |
| データ取得 | `getServerSideProps` / `getStaticProps` | Server Component内で `await fetch` |
| コンポーネント | 全部クライアント（SSR後にhydrate） | 既定 Server Component |
| API | `pages/api/x.ts`（`(req,res)`） | `app/api/x/route.ts`（`Request`/`Response`） |
| スタイル | `styles/` ＋ CSS Modules | `app/globals.css` 等 |

## 基本の書き方（コード）

ページ（`pages/index.tsx`）。default export したコンポーネントがそのまま画面。
```tsx
// pages/index.tsx → "/"
export default function Home() {
  return <h1>こんにちは Pages Router</h1>;
}
```

全ページ共通ラッパー（`_app.tsx`）。Layout や Provider をここで。
```tsx
// pages/_app.tsx
import type { AppProps } from "next/app";
import "../styles/globals.css";

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}
```

サーバ側データ取得（リクエスト毎にSSR）。
```tsx
// pages/users.tsx
export const getServerSideProps = async () => {
  const users = await fetch("https://api.example.com/users").then((r) => r.json());
  return { props: { users } };        // ← props として画面に渡る
};
export default function Users({ users }) {
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

API ルート（`pages/api/*`）。`(req, res)` 形式。
```ts
// pages/api/hello.ts → "/api/hello"
import type { NextApiRequest, NextApiResponse } from "next";
export default function handler(req: NextApiRequest, res: NextApiResponse) {
  res.status(200).json({ message: "hello" });
}
```

動的ルート（`[slug].tsx`）と静的生成。
```tsx
// pages/blog/[slug].tsx → "/blog/xxx"
export const getStaticPaths = async () => ({
  paths: [{ params: { slug: "first" } }],
  fallback: false,
});
export const getStaticProps = async ({ params }) => ({
  props: { slug: params.slug },
});
export default function Post({ slug }) {
  return <h1>{slug}</h1>;
}
```

## 実務での使い方・定番パターン
- **データ取得の使い分け**：毎回最新→`getServerSideProps`（SSR）、ビルド時に固定→`getStaticProps`（SSG）、定期再生成→`getStaticProps` + `revalidate`（ISR）。
- **`_document.tsx`** は `<html>`/`<body>` のカスタムや `<head>` への要素追加に使う（任意）。
- **CSS Modules**（`Home.module.css`）でスコープ付きCSS。グローバルは `styles/globals.css`（`_app.tsx` で読む）。
- 基本ツールは **`next/link` / `next/image` / `next/router`（`useRouter`）**。App Router の `next/navigation` とは別物。

## ハマりどころ / アンチパターン
- **App Router の作法を持ち込む**：`getServerSideProps` は App Router（`app/`）では使えない。逆に App Router の `"use client"`/`layout.tsx` は Pages Router にない。混同しない。
- **`pages/` と `app/` の併存**：同一プロジェクトで両方使えるが、同じパスを両方に作ると衝突する。
- **`_document.tsx` の必須構造**：カスタムする場合 `<Html><Head /><body><Main /><NextScript /></body></Html>` を崩すと壊れる。
- **`getInitialProps`（最古の方式）**：今は基本使わない（SSGの恩恵を消す）。`getServerSideProps`/`getStaticProps` を使う。
- **クライアント専用コードをトップレベルで実行**：`window` 参照は `useEffect` 内へ（SSR時に `window is not defined`）。

## フォルダ構成（始動直後）

> `create-next-app` で App Router を選ばなかった場合の構成。`pages/` の最小セット＋設定。
> **ルート・コンポーネント・API は自分で `pages/` 配下に足していく**。

```
myapp/
├── pages/                            # ファイル＝URL（Pages Router）
│   ├── _app.tsx                      # 全ページ共通ラッパー（Layout/Provider）【生成】
│   ├── _document.tsx                 # <html><body>のカスタム（任意・自分で作る）
│   ├── index.tsx                     # "/" の画面【生成】
│   ├── about.tsx                     # "/about"（自分で作る）
│   ├── blog/
│   │   ├── index.tsx                 # "/blog"
│   │   └── [slug].tsx                # "/blog/xxx"（動的ルート）
│   └── api/                          # APIルート（サーバ関数）
│       └── hello.ts                  # "/api/hello"（(req,res)=>{}）【生成: hello.js】
├── public/                           # 静的ファイル（"/" から配信）【生成】
│   └── favicon.ico  vercel.svg
├── styles/                           # CSS【生成】
│   ├── globals.css                   # グローバルCSS（_app.tsx で読む）
│   └── Home.module.css               # CSS Modules（スコープ付き）
├── components/                       # 共通コンポーネント（自分で作る）
├── lib/                              # ユーティリティ（自分で作る）
├── next.config.js                    # Next 設定【生成】
├── tsconfig.json                     # TS設定【生成】（TS選択時）
├── next-env.d.ts                     # Next 生成の型定義（触らない）【生成】
├── package.json                      # scripts: dev/build/start/lint【生成】
├── .gitignore                        # 【生成】
└── node_modules/                     # 依存の実体
```
- **`pages/` のファイル＝URL**。`pages/about.tsx`→`/about`、`pages/blog/[slug].tsx`→`/blog/xxx`。
- `_app.tsx`＝全ページ共通の枠（App Routerの `layout.tsx` 相当）、`_document.tsx`＝`<html>`構造のカスタム。
- `pages/api/*` は **APIルート**（`export default function handler(req, res)`）。App Routerの `route.ts` とは別物。
- **【生成】= `create-next-app`（App Router を選ばなかった場合）。** `components/`・`lib/`・追加ルートは自分で作る。

## 関連
[../nextjs15/getting_started.md](../nextjs15/getting_started.md)（App Router・現行）
