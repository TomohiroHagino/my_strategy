# 実務でハマる罠まとめ（Next.js 12・Pages Router）

## ひとことで言うと
Pages Router 特有の落とし穴を1か所に集約——App Router の作法混入、`window is not defined`、`_document` の構造崩し、`getInitialProps` の弊害、`pages`/`app` 併存衝突など、案件で繰り返し踏む罠と回避策。

## 役割・なぜ必要か
- 各トピックのファイルにも罠は書いてあるが、**実装中に「これ前も踏んだ」と思った時の早見表**として横断的にまとめる。
- 多くは「App Router の知識をそのまま持ち込む」「サーバ実行とクライアント実行を取り違える」の2系統に集約される。原因の型を覚えると予防できる。

## 罠リスト（症状 → 原因 → 回避）

### 1. App Router の作法を持ち込む
- **症状**：`getServerSideProps`/`getStaticProps` が App Router で効かない、`"use client"`/`layout.tsx`/`metadata` を Pages Router で書いても無視される。
- **原因**：両者は別アーキテクチャ。データ取得関数・共通枠・メタの仕組みが全く違う。
- **回避**：Pages Router は取得関数（`getStaticProps`等）＋`_app`/`_document`＋`next/head`。App Router の機能（RSC・Server Actions・`app/`）は**存在しない**前提で書く（→ [../nextjs15/](../nextjs15/)）。

### 2. `window is not defined`（SSR時）
- **症状**：`ReferenceError: window is not defined` でサーバ生成が落ちる。
- **原因**：`window`/`document`/`localStorage` をコンポーネント本体やモジュールトップレベルで参照。サーバには存在しない。
- **回避**：ブラウザ専用コードは `useEffect` 内へ。または `typeof window !== "undefined"` でガード。重いブラウザ専用ライブラリは `next/dynamic` で `ssr: false` にして読む。

```tsx
import dynamic from "next/dynamic";
const Chart = dynamic(() => import("../components/Chart"), { ssr: false });
```

### 3. ハイドレーション不一致
- **症状**：`Hydration failed`／初回だけ表示が崩れる/ちらつく。
- **原因**：サーバHTMLとクライアント初回描画が食い違う。`Date.now()`/`Math.random()`/ロケール依存の整形/`window` 参照をレンダー本体で使うと発生。
- **回避**：これらは `useEffect` 後に反映するか、サーバ側で値を確定して props で渡す（→ [rendering.md](./rendering.md)）。

### 4. `_document.tsx` の構造崩し
- **症状**：何も描画されない／スクリプトが動かない。
- **原因**：`<Html><Head /><body><Main /><NextScript /></body></Html>` の必須要素を消した/動かした。
- **回避**：この構造を保つ。`_document` はサーバのみ実行で `useEffect`/イベント不可（→ [app_document.md](./app_document.md)）。

### 5. `getInitialProps` の弊害
- **症状**：全ページが毎回サーバ実行になり遅い／SSGが効かない。
- **原因**：`getInitialProps`（特に `_app.tsx` に付与）はサーバ/クライアント両方で走り、静的最適化を無効化する。
- **回避**：`getStaticProps`/`getServerSideProps` へ移行。`getInitialProps` はレガシー保守以外で新規採用しない（→ [data_fetching.md](./data_fetching.md)）。

### 6. `pages/` と `app/` の併存衝突
- **症状**：同じパスが二重定義でビルド警告/予期せぬ画面。
- **原因**：13〜15 で両Routerは併存できるが、同一パスを両方に作ると衝突する。
- **回避**：パスの担当を片方に決める。移行中は重複させない。

### 7. global CSS の import 場所
- **症状**：`Global CSS cannot be imported from files other than your Custom <App>`。
- **原因**：`_app.tsx` 以外で global CSS を import した。
- **回避**：global は `_app.tsx` でだけ。スコープが要るものは `*.module.css`（→ [styling.md](./styling.md)）。

### 8. `router.query` が初回空
- **症状**：動的ルートで初回レンダー時に `query` の値が `undefined`。
- **原因**：静的最適化や `fallback` 中は `query` が未確定。
- **回避**：`router.isReady` を見るか、`getStaticProps`/`getServerSideProps` の `params` で受ける（→ [routing.md](./routing.md)）。

### 9. `next/navigation` を import
- **症状**：`useRouter`/`usePathname` が動かない・エラー。
- **原因**：それは App Router 用。Pages Router は `next/router`。
- **回避**：Pages Router では `import { useRouter } from "next/router"`（→ [routing.md](./routing.md)）。

### 10. API ルートの `res` 二重送信
- **症状**：`Cannot set headers after they are sent to the client`。
- **原因**：分岐後に `return` せず、複数回 `res` を送った。
- **回避**：各分岐で `return res...`。メソッド未対応は `405`（→ [api_routes.md](./api_routes.md)）。

### 11. `next export` で動的機能を期待
- **症状**：`export`（Next 12）後に SSR/API/ISR/ミドルウェアが動かない。
- **原因**：`export` は完全静的化。
- **回避**：動的機能が要るなら `next start`/Vercel/standalone（→ [deployment.md](./deployment.md)）。

### 12. 秘密に `NEXT_PUBLIC_` を付ける
- **症状**：APIキー等がブラウザバンドルから露出。
- **原因**：`NEXT_PUBLIC_*` はクライアントに焼き込まれる。
- **回避**：秘密は接頭辞なし＝サーバ専用。公開してよい値だけ `NEXT_PUBLIC_`（→ [deployment.md](./deployment.md)）。

## 関連
[routing.md](./routing.md) / [data_fetching.md](./data_fetching.md) / [rendering.md](./rendering.md) / [api_routes.md](./api_routes.md) / [app_document.md](./app_document.md) / [styling.md](./styling.md) / [deployment.md](./deployment.md)
