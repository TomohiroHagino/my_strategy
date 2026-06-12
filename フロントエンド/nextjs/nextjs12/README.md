# Next.js 12（Pages Router）リファレンス

> **この版 = Pages Router 時代（〜Next.js 12 が既定）**。App Router 登場前の従来方式。
> 現行（App Router）は [../nextjs15/](../nextjs15/) を参照。Pages Router は 13〜15 でも併存・サポートされる。

## この版のポイント（App Router との違い）
- **ルーティング**：`pages/` のファイル＝URL（`app/` フォルダではない）。
- **データ取得**：`getServerSideProps`（SSR）/ `getStaticProps`＋`getStaticPaths`（SSG/ISR）。
- **共通枠**：`_app.tsx` / `_document.tsx`（`layout.tsx` ではない）。
- **API**：`pages/api/*` の `(req, res)` 形式。
- コンポーネントは全部クライアント（React Server Components ではない）。

## 各ファイルの書式（テンプレ）
```markdown
# {トピック名}（Next.js 12・Pages Router）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```

## 項目（各ファイルへ）

### はじめに / ルーティング
- [getting_started.md](./getting_started.md) … 始め方・フォルダ構成・App Routerとの違いの俯瞰
- [vs_app_router.md](./vs_app_router.md) … **App Router（nextjs15）との違い**・コード対比・移行ポイント（徹底比較）
- [routing.md](./routing.md) … ファイルベースルーティング（`pages/`・動的`[id]`・catch-all・`<Link>`・`useRouter`）

### 描画 / データ取得
- [request_flow.md](./request_flow.md) … リクエストの流れ（middleware→pages→getServerSideProps→HTML）・各部分が何を返すか
- [rendering.md](./rendering.md) … 描画戦略（SSG / SSR / ISR / CSR）の使い分けとハイドレーション
- [data_fetching.md](./data_fetching.md) … `getStaticProps`/`getStaticPaths`/`getServerSideProps`/ISR/`getInitialProps`/クライアント取得

### 共通枠 / API / メタ
- [app_document.md](./app_document.md) … `_app.tsx`（共通ラッパー/global CSS/Provider/getLayout）と `_document.tsx`（html構造）
- [api_routes.md](./api_routes.md) … `pages/api`（`(req,res)`ハンドラ・`req.method`分岐・動的API・`req.body`/`query`）
- [metadata_seo.md](./metadata_seo.md) … `next/head` の `<Head>`・title/meta/OGP・`next/script`
- [middleware.md](./middleware.md) … `middleware.ts`（Edge・`NextResponse`・`matcher`・認証/リダイレクト）

### スタイル / 運用 / 罠
- [styling.md](./styling.md) … global CSS / CSS Modules / styled-jsx / Sass / Tailwind
- [deployment.md](./deployment.md) … `next build`/`start`・`output:'standalone'`・Vercel・`next export`・環境変数
- [pitfalls.md](./pitfalls.md) … Pages Router 実務の罠（App Router混同・`window is not defined`・`_document`構造 等）

> レガシー保守向けの要点まとめ。新規開発は App Router（[nextjs15](../nextjs15/)）推奨。
