# 始め方（React）

## ひとことで言うと
Reactアプリを**新規に作って動かす**ための最初の一歩。現在は **Vite**（高速なビルドツール）でプロジェクトを生成し、TypeScript で書くのが定番。

## 役割・なぜ必要か
- Reactは「UIを部品（コンポーネント）に分け、**状態(state)から画面を組み立てる**」ライブラリ。素のHTML/JSではなく、JSX＋ビルド環境が要るので、足場を用意するツールが必要。
- かつての `create-react-app`（CRA）は**メンテ停止・非推奨**。今は **Vite** が事実上の標準（起動・更新が速く、設定もシンプル）。
- TypeScript 前提（`-ts` テンプレート）にすると、props や state に型がつき、実務での事故が激減する。

## 基本の書き方（コード）
プロジェクト作成（CRAではなく **Vite** を使う）:
```bash
# react-ts テンプレートで生成（npm 7+ では -- が必要）
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev      # 開発サーバ起動（http://localhost:5173）
```

エントリポイント `src/main.tsx`（ここで1回だけDOMにマウント）:
```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import "./index.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

画面本体 `src/App.tsx`（**関数コンポーネント**：JSXを返す関数）:
```tsx
function App() {
  const name = "World";
  // return の中身が JSX（HTMLに似た式）。{} で JS の値を埋め込める
  return (
    <main>
      <h1>Hello, {name}!</h1>
      <p>これが最初のReactコンポーネントです。</p>
    </main>
  );
}

export default App;
```

## 実務での使い方・定番パターン
- **開発は `npm run dev`、本番ビルドは `npm run build`**（`dist/` が出力される）、確認は `npm run preview`。
- 1ファイル＝1コンポーネントを基本に、`src/components/` 以下へ機能ごとに分けて置く。
- `index.html` に `<div id="root"></div>` があり、そこへ `main.tsx` がアプリを差し込む（**マウントは1回だけ**）。
- 型チェックは別途 `tsc --noEmit`、整形は Prettier、静的解析は ESLint を組み合わせるのが実務の定番。
- ルーティングやデータ取得は React に**内蔵されていない**ので、必要なら別ライブラリを足す（→ routing / data_fetching）。最初から全部入りが欲しいなら Next.js も選択肢。

## ハマりどころ / アンチパターン
- **CRA で始めない**：`create-react-app` は非推奨でビルドが遅い。新規は **Vite** 一択（要件によっては Next.js）。
- `npm create vite@latest my-app --template react-ts` のように **`--` を省くと**テンプレ指定が Vite に渡らず対話モードになる。npm 経由では `-- --template react-ts` と書く。
- **`StrictMode` の二重実行**：開発時のみ、コンポーネントの描画や `useEffect` が**2回**走る（副作用バグを炙り出すための仕様）。本番では1回。これを「バグ」と勘違いしない。→ [useeffect.md](./useeffect.md)
- 拡張子は **`.tsx`**（JSXを書くファイル）。`.ts` のままだと JSX が型エラーになる。
- `document.getElementById("root")` の `!`（non-null assertion）は「必ず存在する前提」。`index.html` の id とズレると `null` で落ちる。
- ファイル/コンポーネント名は大文字始まり（`App`）。小文字始まりはHTMLタグ扱いになる。→ [components_jsx.md](./components_jsx.md)

## フォルダ構成（始動直後）
```
myapp/
├── index.html              # エントリHTML（<div id="root"> を持つ）
├── src/
│   ├── main.tsx            # ReactをDOMにマウント（1回だけ）
│   ├── App.tsx             # 画面本体（関数コンポーネント）
│   ├── App.css             # App 用のCSS
│   ├── index.css           # 全体CSS
│   ├── vite-env.d.ts       # Vite が生成する型定義（触らない）
│   └── assets/
│       └── react.svg       # サンプル画像
├── public/
│   └── vite.svg            # そのまま配信される静的ファイル
├── vite.config.ts          # Vite の設定
├── tsconfig.json           # TS設定（参照プロジェクト構成）
├── tsconfig.node.json      # Vite 設定ファイル用の TS 設定
├── eslint.config.js        # ESLint 設定
├── package.json            # scripts: dev / build / preview
├── .gitignore              # node_modules 等を除外
├── node_modules/           # 依存パッケージ（npm install で生成）
└── src/components/         # 機能ごとの部品（# 自分で作る）
```
- Vite作成例。React単体はルーティング等を含まないので `react-router` 等を自分で足す。

## 関連
[components_jsx.md](./components_jsx.md) / [state.md](./state.md)
