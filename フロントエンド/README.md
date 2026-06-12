# フロントエンド 実務リファレンス（索引）

> ブラウザ側（と、それを動かすツール群）を **言語 → ビルドツール → フレームワーク** の順でまとめる索引。
> 各フレームワークは「版ごとに独立したフォルダ」で、`getting_started.md`（深いフォルダツリー付き）と `request_flow.md`（描画/データの流れ）を基本セットで持つ。

## 全体像
```
 言語          JavaScript / TypeScript          … 土台
   │
 ビルド        Vite / esbuild / Rollup / Turbopack … 変換・バンドル・dev server
   │
 フレームワーク  React / Vue（UIライブラリ）
   │           Next.js / Nuxt（その上のフルスタックFW）
   │           Leptos（Rust/WASM の別系統）
   ▼
 ブラウザ（DOM）
```

## まず読む（土台）
- [javascript/README.md](./javascript/README.md) … JavaScript（言語の概要）
- [typescript/README.md](./typescript/README.md) … TypeScript（言語の概要）
- [build_tools.md](./build_tools.md) … ビルドツール（Vite / esbuild / Rollup / Turbopack）。webpackの後継は何か

## フレームワーク（各版へ）

### React 系
- [react/README.md](./react/README.md) … React（概要）
  - [react/react19/](./react/react19/) … **React 18/19** 実務リファレンス（フラッグシップ）
- [nextjs/README.md](./nextjs/README.md) … Next.js（概要）
  - [nextjs/nextjs15/](./nextjs/nextjs15/) … **Next.js 15（App Router）** 実務リファレンス
  - [nextjs/nextjs12/](./nextjs/nextjs12/) … **Next.js 12（Pages Router）** 実務リファレンス

### Vue 系
- [vue/README.md](./vue/README.md) … Vue（概要）
  - [vue/vue3/](./vue/vue3/) … **Vue 3** 実務リファレンス（フラッグシップ）
- [nuxt/README.md](./nuxt/README.md) … Nuxt（概要）
  - [nuxt/nuxt3/](./nuxt/nuxt3/) … **Nuxt 3** 実務リファレンス（フラッグシップ）

### Rust / WASM 系
- [rust/README.md](./rust/README.md) … Rust（フロントエンド用途の概要）
  - [rust/leptos/leptos0/](./rust/leptos/leptos0/) … **Leptos 0.x** 実務リファレンス（フラッグシップ）

## どれを選ぶか（ざっくり）
| やりたいこと | 第一候補 |
|---|---|
| UIだけ・既存に載せる | **React** / **Vue** |
| SSR/SSG・ルーティング・APIまで一体 | **Next.js**（React系）/ **Nuxt**（Vue系） |
| Pages Router の既存案件 | **Next.js 12** |
| Rust/WASMで書きたい | **Leptos** |
| ビルドツール（新規） | **Vite**（迷ったらこれ）→ [build_tools.md](./build_tools.md) |

## 各ファイルの書式（テンプレ）
各FWフォルダ内の項目ファイルは下記テンプレで統一:
```markdown
# {概念名}（{フレームワーク}）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```

## 関連
バックエンド連携: [../バックエンド/](../バックエンド/) ／ 配信・デプロイ: [../DevOps/deploy_strategies.md](../DevOps/deploy_strategies.md)
