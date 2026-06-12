# ルーティング（React Router）（React）

## ひとことで言うと
URL に応じて**表示する画面(コンポーネント)を切り替える**仕組み。**React 本体にルーティングは無い**ので、定番の外部ライブラリ **React Router** を足して実現する。

## 役割・なぜ必要か
- React は「UIを部品に分けて状態から組み立てる」ライブラリで、**URLと画面の対応付けは持っていない**。SPA(単一HTML)でページ遷移っぽい体験を出すにはルーターが要る。
- React Router が `/users/1` のようなURLを見て、**ページを再読み込みせずに**対応コンポーネントを描画する(クライアントサイドルーティング)。
- 戻る/進む・ブックマーク・URL直打ちといった「Webの当たり前」を SPA でも成立させる。

## 基本の書き方（コード）
```tsx
// main.tsx — 全体を BrowserRouter で包む
import { BrowserRouter } from "react-router-dom";
import { createRoot } from "react-dom/client";
import { App } from "./App";

createRoot(document.getElementById("root")!).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

```tsx
// App.tsx — Routes / Route で URL とコンポーネントを対応付け
import { Routes, Route, Link } from "react-router-dom";

export function App() {
  return (
    <>
      <nav>
        {/* <a> でなく Link を使う（リロードせず遷移） */}
        <Link to="/">ホーム</Link>
        <Link to="/users/1">ユーザー1</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/users/:id" element={<UserPage />} /> {/* :id は動的 */}
        <Route path="*" element={<NotFound />} />          {/* 404 */}
      </Routes>
    </>
  );
}

function Home() { return <h1>ホーム</h1>; }
function NotFound() { return <h1>404 Not Found</h1>; }
```

```tsx
// UserPage.tsx — useParams で URL のパラメータ、useNavigate で遷移
import { useParams, useNavigate } from "react-router-dom";

function UserPage() {
  const { id } = useParams();          // /users/1 → id === "1"
  const navigate = useNavigate();      // コードから遷移する関数

  return (
    <div>
      <h1>ユーザー {id}</h1>
      <button onClick={() => navigate("/")}>ホームへ戻る</button>
      <button onClick={() => navigate(-1)}>1つ戻る</button>
    </div>
  );
}
```

## 実務での使い方・定番パターン
- **`BrowserRouter`** … アプリ全体を一度だけ包むルートの土台。
- **`Routes` / `Route`** … URLパターン(`path`)と表示要素(`element`)の対応表。上から評価され**最初に一致したもの**を描画。
- **`Link`** … 画面内リンク。`<a href>` と違い**リロードせず**遷移する。
- **`useNavigate`** … ボタン押下やAPI成功後など、**コードから**遷移したいとき(`navigate("/done")`)。
- **`useParams`** … `/users/:id` の `:id` 部分を取り出す。詳細ページの定番。
- **ネスト/共通レイアウト** … 親 `Route` に `element={<Layout/>}`、子で `<Outlet/>` を置くと共通ヘッダ＋切替本文が作れる。
- 取得したパラメータでデータ取得につなぐのが定番。→ [data_fetching.md](./data_fetching.md)

## ハマりどころ / アンチパターン
- **`<a href>` で内部遷移してしまう** → ページ全体がリロードされ state が消える。内部遷移は必ず `Link`(または `navigate`)。
- React Router は**外部ライブラリ前提**。`npm i react-router-dom` を忘れると当然動かない。バージョンでAPIが変わる(v6/v7系)点にも注意。
- `BrowserRouter` で包み忘れると `useNavigate`/`useParams` が**エラー**になる。
- 本番のサーバ設定で**全URLを index.html に返す(SPAフォールバック)**を入れないと、URL直打ち/リロードで 404 になる。
- **Next.js を使うならルーティングは内蔵**(ファイルベース)なので React Router は不要。→ [../nextjs/](../../nextjs/nextjs15/)

## 関連
[data_fetching.md](./data_fetching.md)
