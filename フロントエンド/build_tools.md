# フロントエンドのビルドツール（Vite / esbuild / Rollup / Turbopack）

## ひとことで言うと
ソースの **TS/JSX/CSS/画像をブラウザが読める形に変換し、1つ（数個）にまとめる**仕組み。いまの新規開発は **Vite が事実上の標準**で、webpack はレガシー扱いになりつつある。内部は Go製 **esbuild** / Rust製 **SWC・Turbopack** など「ネイティブ言語製の高速ツール」への世代交代が進行中。

## 役割・なぜ必要か
ブラウザは素のままでは TS・JSX・`import` 解決・古いブラウザ対応・CSS変換などをしてくれない。ビルドツールが以下を担う:
- **トランスパイル**: TS/JSX → JS、新しい構文 → 古いブラウザ向けに変換
- **バンドル**: 大量の `import` を解決し、配信に適したファイルへまとめる
- **最適化**: Tree-shaking（未使用コード削除）・minify・コード分割・ハッシュ付与
- **開発サーバ（dev）**: ファイル変更を即反映する **HMR（ホットリロード）**

開発体験（dev）と本番出力（build）の**両方の速さ**が選定の決め手。ここで webpack が遅くて敬遠され、Vite に置き換わった。

## 主要ツールの地図
```
                  用途
  アプリ開発 ─────────────────────────── ライブラリ公開
     │                                        │
   Vite ⭐（dev=esbuild / build=Rollup）     Rollup / tsup
     │                                        │
   Turbopack（Next.js純正・Rust）           esbuild（薄いラッパ）

  エンジン層（ツールの“中身”）:
   esbuild(Go)  … 事前バンドル・トランスパイル超高速
   SWC(Rust)    … Babel代替トランスパイラ（Next.jsが採用）
   Rollup(JS)   … tree-shakingが綺麗。Viteの本番担当
   Rolldown(Rust) … Rollup後継。将来Viteの本番もこれへ
```

| ツール | 製作言語 | 位置づけ | 速さの理由 |
|---|---|---|---|
| **Vite** ⭐ | JS（中身はGo/Rust） | 新規アプリの第一候補。Vue/React/Svelte/Solid 対応 | dev=esbuildで事前バンドル＋ネイティブESM、build=Rollup |
| **esbuild** | Go | 超高速バンダラ/トランスパイラ。Viteの内部エンジン | Go並列処理でwebpackの10〜100倍 |
| **Rollup** | JS | ライブラリ向け定番＆Viteの本番ビルド担当 | 出力が素直でtree-shakingが綺麗 |
| **Turbopack** | Rust | Next.js純正の次世代バンダラ | Rust製・インクリメンタル |
| **Rspack** | Rust | webpack互換の高速版 | 既存webpack設定をほぼ流用して高速化 |
| **webpack** | JS | レガシー。既存大規模案件で現役 | （遅い。新規では非推奨） |
| **SWC** | Rust | Babel代替トランスパイラ | Next.jsが標準採用 |

## 基本の書き方（コード）
Vite プロジェクトの作成と起動:
```bash
npm create vite@latest my-app -- --template react-ts
cd my-app && npm install
npm run dev     # 開発サーバ（HMR）
npm run build   # 本番ビルド（dist/ に出力）
npm run preview # 本番ビルドをローカル確認
```
`vite.config.ts`（最小〜よく足す設定）:
```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'node:path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: { '@': path.resolve(__dirname, 'src') }, // import '@/components/...'
  },
  server: {
    port: 5173,
    proxy: { '/api': 'http://localhost:8080' }, // dev中のCORS回避
  },
  build: {
    sourcemap: true,
    rollupOptions: {
      output: { manualChunks: { vendor: ['react', 'react-dom'] } }, // 分割
    },
  },
})
```
Next.js は専用ツールなので個別設定は不要。Turbopack を使うだけ:
```bash
next dev --turbo   # 開発をTurbopackで（webpackより高速）
```

## 実務での使い方・定番パターン
- **新規アプリは Vite 一択でよい**。Vue/React/Svelte/Solid どれでもテンプレがある。
- **Next.js / Nuxt を使うならビルドツールは意識しない**。フレームワーク同梱（Next=webpack→Turbopack移行中、Nuxt=Vite）。
- **npm に出すライブラリは Rollup か tsup**（tsup＝esbuildの薄いラッパで設定ゼロに近い）。アプリ用のViteとは目的が違う（バンドルを小さく・型定義同梱・CJS/ESM両対応）。
- **環境変数**: Vite は `import.meta.env.VITE_*` のみクライアントに露出（`VITE_` 接頭辞が必須）。秘密情報を素で埋め込まない。→ [../../インフラ/](../../インフラ/)
- **テストランナーと統一**: Vite製の **Vitest** を使うと設定（alias/plugin）を共有できる。→ 各FWの [react/react19/testing.md](./react/react19/testing.md) 等
- **既存webpack案件の高速化**: 全面移行が重いなら **Rspack** が設定流用しやすい。

## ハマりどころ / アンチパターン
- **dev は動くのに build で壊れる**: dev はネイティブESMで都度変換、build は Rollup でバンドル、と**経路が違う**ため。リリース前に必ず `npm run build && npm run preview` で確認。
- **環境変数が `undefined`**: Vite は `VITE_` 接頭辞のない変数をクライアントへ渡さない。`process.env.XXX` の癖で書くと空になる（`import.meta.env.VITE_XXX` を使う）。
- **CommonJS ライブラリでエラー**: Vite はESM前提。古いCJS専用パッケージは `optimizeDeps` / `ssr.noExternal` の調整が要ることがある。
- **webpack 用設定をそのまま持ち込む**: `webpack.config.js` の loader/plugin はViteには無効。Viteプラグインに置き換える必要がある。
- **Turbopack はまだ機能差がある**: Next.js のwebpack向け設定・一部プラグインが未対応な場合あり。問題が出たら `--turbo` を外して切り分ける。
- **「とりあえずwebpack」**: 新規で選ぶ理由はほぼ無い（遅い・設定が重い）。レガシー維持以外は避ける。

## 関連
各FWの始め方: [react/react19/getting_started.md](./react/react19/getting_started.md) / [vue/vue3](./vue/vue3/) / [nextjs/nextjs15](./nextjs/nextjs15/) / [nuxt/nuxt3](./nuxt/nuxt3/)
テスト基盤（Vite製のVitest）: [react/react19/testing.md](./react/react19/testing.md)
デプロイ/配信: [../DevOps/deploy_strategies.md](../DevOps/deploy_strategies.md)
