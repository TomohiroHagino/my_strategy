# はじめに（getting started）（Next.js 15）

## ひとことで言うと
`create-next-app` で雛形を作り、`app/` ディレクトリに `page.tsx` / `layout.tsx` を置いて画面を組み、`npm run dev` で開発サーバを立てる——これが **Next.js 15（App Router）の出発点**。

## 役割・なぜ必要か
- Next.js は **React の上に乗るフルスタックフレームワーク**。ルーティング・SSR/SSG・データ取得・APIまで内蔵し、「ビルド設定やルータを自前で組む」手間を省く。
- `create-next-app` が **TypeScript・ESLint・Tailwind・App Router** をまとめて初期設定してくれるので、最初の1コマンドで「動く土台」が手に入る。
- App Router は **React Server Components（RSC）が既定**。最初の構成選択が後の書き方（サーバ/クライアントの分け方）を決めるので、ここを外さないことが重要。

## 基本の書き方（コード）
```bash
# 対話式に雛形を作る（@latest で最新版を取得）
npx create-next-app@latest my-app

# 主な質問（推奨の選択）
# ✔ TypeScript?            → Yes
# ✔ ESLint?               → Yes
# ✔ Tailwind CSS?         → Yes
# ✔ src/ directory?       → 好み（Yes だと src/app/ になる）
# ✔ App Router?           → Yes（★これが本リファレンスの前提）
# ✔ Turbopack for dev?    → Yes（高速な開発サーバ）

cd my-app
npm run dev   # http://localhost:3000 で確認
```

```text
my-app/
├─ app/
│  ├─ layout.tsx        # 全ページ共通の外枠（<html><body>）。必須
│  ├─ page.tsx          # "/" の画面
│  └─ globals.css       # 全体CSS（Tailwindの読み込み等）
├─ public/              # そのまま配信される静的ファイル（画像など）
├─ next.config.ts       # Next の設定
├─ tsconfig.json        # TS設定（"@/*" のパスエイリアス等）
└─ package.json         # scripts: dev / build / start / lint
```

```tsx
// app/layout.tsx — すべてのページを包む“ルートレイアウト”
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = { title: "My App" };

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body>{children}</body>
    </html>
  );
}
```

```tsx
// app/page.tsx — "/" にアクセスした時に表示される画面（Server Component）
export default function Home() {
  return <h1 className="text-2xl font-bold">こんにちは Next.js 15</h1>;
}
```

## 実務での使い方・定番パターン
- **`npm run dev`** で開発、**`npm run build` → `npm run start`** で本番相当の確認。`build` が通るかは常に意識する（型エラー・未使用変数で落ちることがある）。
- **`app/` 配下のファイル名は規約**。`page.tsx`＝画面、`layout.tsx`＝外枠、`loading.tsx`＝読み込み中、`error.tsx`＝エラー画面。名前が役割そのもの。→ [app_router.md](./app_router.md)
- **`@/` パスエイリアス**（`tsconfig.json` の `paths`）で `@/components/Button` のように import できる。相対パス地獄を避ける。
- **コンポーネントは既定でサーバ実行**。`useState` などブラウザ機能が要る部品の先頭にだけ `"use client"` を書く。→ [server_client_components.md](./server_client_components.md)
- **Tailwind** を入れたなら `globals.css` の `@import "tailwindcss";`（v4系）を確認。クラスで即スタイリングできる。

## ハマりどころ / アンチパターン
- **App Router と Pages Router の取り違え**：`app/` と `pages/` は別物。本リファレンスは **App Router 前提**。古い記事の `pages/index.tsx` や `getServerSideProps` をそのまま持ち込まない（App Router では使わない）。
- **`page.tsx` を置き忘れる**：フォルダを作っただけではルートにならない。そのパスを画面にするには `page.tsx` が必須。→ [app_router.md](./app_router.md)
- **ルートレイアウトの `<html>`/`<body>` 欠落**：`app/layout.tsx` には `<html>` と `<body>` が必須。消すとビルド/実行で警告・エラー。
- **`create-next-app` の途中質問を流し読み**：App Router を選ばないと全く別構成になる。最初の選択が一番大事。
- **`"use client"` を入れすぎ**：全部クライアント化するとRSCの利点（サーバでのデータ取得・小さいJS）が消える。必要な葉の部品だけに留める。
- **`node_modules` をコミット**：`.gitignore` に入っているのが既定。手で消さない。

## フォルダ構成（始動直後）

> `create-next-app` が作るのは `app/` の最小セット＋設定ファイル。
> **ルート（フォルダ）・コンポーネント・API は自分で `app/` 配下に足していく**。

```
myapp/
├── app/                              # App Router（フォルダ＝URL）
│   ├── layout.tsx                    # ルートレイアウト（<html><body>）必須【生成】
│   ├── page.tsx                      # "/" の画面【生成】
│   ├── globals.css                   # 全体CSS（Tailwind読み込み）【生成】
│   ├── favicon.ico                   # ファビコン【生成】
│   ├── loading.tsx                   # 読み込み中UI（自分・任意）
│   ├── error.tsx                     # エラーUI（"use client"・自分・任意）
│   ├── not-found.tsx                 # 404画面（自分・任意）
│   ├── about/
│   │   └── page.tsx                  # "/about"（自分で作る）
│   ├── blog/
│   │   ├── page.tsx                  # "/blog"（一覧）
│   │   └── [slug]/                   # 動的セグメント
│   │       └── page.tsx              # "/blog/xxx"（記事ページ）
│   ├── (marketing)/                  # ルートグループ＝URLに出ない整理用（任意）
│   └── api/
│       └── users/
│           └── route.ts              # APIエンドポイント（GET/POST関数＝Route Handler）
├── components/                       # 共通コンポーネント（自分で作る）
│   └── ui/Button.tsx
├── lib/                              # DBクライアント・ユーティリティ（自分で作る）
│   └── db.ts
├── public/                           # "/" から配信される静的ファイル【生成】
│   └── next.svg  vercel.svg          # サンプル画像
├── middleware.ts                     # 認証/リダイレクト（自分・任意・ルート直下に置く）
├── next.config.ts                    # Next 設定【生成】
├── tsconfig.json                     # TS設定（"@/*" エイリアス）【生成】
├── next-env.d.ts                     # Next 生成の型定義（触らない）【生成】
├── eslint.config.mjs                 # ESLint【生成】
├── postcss.config.mjs                # PostCSS（Tailwind）【生成】
├── package.json                      # scripts: dev/build/start/lint【生成】
├── .gitignore                        # 【生成】
└── node_modules/                     # 依存の実体（npm install）
```
- **App Router：`app/` のフォルダ＝URL**。`page.tsx`＝画面、`layout.tsx`＝共通枠。`loading`/`error`/`not-found` は名前が役割の特殊ファイル。
- `[slug]`＝動的ルート、`(marketing)`＝ルートグループ（URLに出ない整理用）、`api/.../route.ts`＝APIエンドポイント。
- **【生成】= `create-next-app`。** `components/`・`lib/`・各ルートのフォルダ・`middleware.ts` は自分で作る。作成時に src/ を選ぶと `src/app/` 構成。
- 既定は Server Component。ブラウザ機能（useState等）が要る葉だけ先頭に `"use client"`。

## 関連
[app_router.md](./app_router.md) / [server_client_components.md](./server_client_components.md)
