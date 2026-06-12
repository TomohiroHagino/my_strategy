# App Router との違い（Pages Router ↔ App Router）（Next.js 12 ↔ 15）

## ひとことで言うと
**Pages Router（旧・本書 nextjs12）** は「`pages/` のファイル＝URL ＋ 専用データ取得関数（`getServerSideProps` 等）＋ 全部クライアントReact」。
**App Router（新・[nextjs15](../nextjs15/)）** は「`app/` のフォルダ＝URL ＋ React Server Components ＋ `fetch` 直書き」。
**ルーティング・データ取得・コンポーネントモデルが根本から違う**。同一プロジェクトでの併存も可能。

## 役割・なぜ必要か
- 既存案件は Pages Router が多い一方、新規は App Router が標準。**両者の対応関係を把握しておくと、移行や混在で迷わない**。
- 「App Router の記事のやり方をそのまま Pages Router に持ち込む（またはその逆）」が最大の事故。違いを1枚で押さえる。

## 全体比較表

| 項目 | Pages Router（旧・nextjs12） | App Router（新・nextjs15） |
|---|---|---|
| ルート定義 | `pages/` の**ファイル**＝URL | `app/` の**フォルダ**＝URL |
| 画面ファイル | `pages/about.tsx` | `app/about/page.tsx` |
| 共通レイアウト | `_app.tsx` / `_document.tsx`（全体で1組） | `layout.tsx`（**ネスト可・フォルダ単位**） |
| データ取得 | `getServerSideProps` / `getStaticProps` / `getStaticPaths` | **Server Component 内で `await fetch`** |
| レンダリング指定 | **どの関数を使うか**で決まる（SSR/SSG/ISR） | 既定SSR＋`fetch`のキャッシュ設定で制御 |
| コンポーネント | 全部クライアント（SSR後に hydrate） | **既定 Server Component**＋`"use client"` |
| データ更新 | API Route ＋ クライアント `fetch` | **Server Actions**（`"use server"`） |
| API | `pages/api/x.ts`（`(req, res)`） | `app/api/x/route.ts`（`Request`/`Response`） |
| メタデータ/SEO | `next/head` の `<Head>` | `metadata` export / `generateMetadata` |
| ローディング/エラー | 自前で実装 | `loading.tsx` / `error.tsx`（規約ファイル） |
| ルーター | `next/router`（`useRouter`） | `next/navigation`（`useRouter`/`usePathname`） |
| リンク | `next/link`（12.0は `<a>` 子要素必須） | `next/link`（`<a>` 不要） |

## 主要な違い（コードで対比）

### ① ページの作り方
```tsx
// 【Pages Router】pages/about.tsx → "/about"
export default function About() { return <h1>About</h1>; }
```
```tsx
// 【App Router】app/about/page.tsx → "/about"
export default function About() { return <h1>About</h1>; }
```
→ 画面の中身は同じだが、**置き場所が「ファイル」か「フォルダ＋page.tsx」か**が違う。

### ② データ取得（リクエスト毎）
```tsx
// 【Pages Router】専用関数 getServerSideProps で取り、props で渡す
export const getServerSideProps = async () => {
  const data = await fetch("https://api.example.com/x").then((r) => r.json());
  return { props: { data } };
};
export default function Page({ data }) { return <pre>{JSON.stringify(data)}</pre>; }
```
```tsx
// 【App Router】Server Component を async にして直接 await
export default async function Page() {
  const data = await fetch("https://api.example.com/x").then((r) => r.json());
  return <pre>{JSON.stringify(data)}</pre>;
}
```
→ App Router は **props のバケツリレーが消える**。`getServerSideProps` は App Router では使えない。

### ③ 共通レイアウト
```tsx
// 【Pages Router】pages/_app.tsx（全ページ共通・1枚）
export default function App({ Component, pageProps }) {
  return <Layout><Component {...pageProps} /></Layout>;
}
```
```tsx
// 【App Router】app/layout.tsx（フォルダ単位でネスト可）
export default function Layout({ children }) {
  return <html><body>{children}</body></html>;
}
```
→ App Router は **ルートごとに入れ子のレイアウト**が作れる（Pages Router は `getLayout` パターンで擬似的に対応）。

### ④ API
```ts
// 【Pages Router】pages/api/users.ts
export default function handler(req, res) { res.status(200).json({ ok: true }); }
```
```ts
// 【App Router】app/api/users/route.ts
export async function GET() { return Response.json({ ok: true }); }
```
→ `(req,res)` か、Web標準の `Request`/`Response` か。

### ⑤ メタデータ（SEO）
```tsx
// 【Pages Router】next/head
import Head from "next/head";
<Head><title>About</title></Head>
```
```tsx
// 【App Router】metadata export
export const metadata = { title: "About" };
```

## 実務での使い分け
- **新規開発 → App Router（nextjs15）**。RSCで初期JSが軽い・SEO/データ取得が素直。
- **既存・レガシー保守 → Pages Router（本書）**。無理に移行しない。
- **併存可能**：同一プロジェクトで `pages/` と `app/` を両方持てる（段階移行に使う）。ただし**同じパスを両方に作ると衝突**する。

## 移行のポイント（Pages → App）
| 旧（Pages） | 新（App）への置き換え |
|---|---|
| `pages/x.tsx` | `app/x/page.tsx` |
| `getServerSideProps` | async Server Component で `await fetch`（`cache: 'no-store'`） |
| `getStaticProps` | Server Component で `fetch`（既定キャッシュ）／`generateStaticParams` |
| `_app.tsx` / `_document.tsx` | `app/layout.tsx`（ルート） |
| `next/router` | `next/navigation` |
| `next/head` | `metadata` / `generateMetadata` |
| `pages/api/x.ts` | `app/api/x/route.ts` |
| クライアント `fetch` で更新 | Server Actions |

## ハマりどころ / アンチパターン
- **やり方の混同**：`getServerSideProps` を `app/` に書く、`"use client"` を `pages/` に期待する——どちらも動かない。**ファイルがどちらのルーター配下かで作法が決まる**。
- **「App Routerの記事」をPages Routerにコピペ**：`layout.tsx`・`loading.tsx`・`route.ts`・`metadata` は Pages Router に存在しない。
- **段階移行で同一URL重複**：`pages/about.tsx` と `app/about/page.tsx` を同時に置くと衝突。
- **`next/router` と `next/navigation` の取り違え**：import 元が違う。Pages は `next/router`。

## 関連
[getting_started.md](./getting_started.md) / [routing.md](./routing.md) / [data_fetching.md](./data_fetching.md) / [../nextjs15/](../nextjs15/)（App Router 現行）
