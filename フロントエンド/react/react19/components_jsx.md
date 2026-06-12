# コンポーネントと JSX（React）

## ひとことで言うと
**コンポーネント**＝「JSXを返す関数」で作るUIの部品。**JSX**＝JavaScriptの中にHTML風の見た目を書ける構文。Reactの基本単位はこの2つ。

## 役割・なぜ必要か
- Reactの核は **UI = f(state)**（画面は状態の関数）。その「f」＝**画面を組み立てる関数**がコンポーネント。
- UIを小さな部品に分けると、再利用・テスト・修正がしやすくなる。ボタン・カード・ヘッダーなどを部品化し、**組み合わせて（合成して）**画面を作る。
- JSXがあるおかげで「どんなDOMになるか」をコード上で直感的に書ける。最終的には Vite/Babel が `React.createElement(...)` 相当に変換する（＝JSXは**式**）。

## 基本の書き方（コード）
関数コンポーネントは **大文字始まり** で、JSXを返す:
```tsx
// 大文字始まり（Button）。小文字だと HTML タグ扱いになる
function Button() {
  return <button>送信</button>;
}

function Card() {
  const title = "お知らせ";
  const count = 3;
  return (
    // ルート要素は1つ。複数並べたいときは Fragment <>...</> で包む
    <>
      {/* class ではなく className。{} で JS の式を埋め込む */}
      <div className="card">
        <h2>{title}</h2>
        <p>未読 {count} 件</p>
        {/* 要素は必ず閉じる（自己終了タグは /> ） */}
        <img src="/icon.png" alt="アイコン" />
        <Button /> {/* コンポーネントを部品として配置（合成） */}
      </div>
    </>
  );
}

export default Card;
```

JSXの規則（最重要）:
```tsx
// 条件は三項演算子か && で（if文は JSX の中に直接書けない）
{isOpen ? <p>開いています</p> : <p>閉じています</p>}
{count > 0 && <span>{count}件</span>}
```

## 実務での使い方・定番パターン
- **1ファイル＝1コンポーネント**を基本に、`src/components/feature/Xxx.tsx` のように機能ごとへ分割。小さく保つ。
- **合成（composition）**：小さな部品（`Button`/`Avatar`）を組み合わせて大きな部品（`Header`/`Card`）を作る。継承ではなく合成がReact流。
- 部品間でデータを渡すときは **props**（→ [props.md](./props.md)）、見た目だけ差し替えたいときは `children` で中身を流し込む。
- 純粋に保つ：同じ入力（props/state）なら同じJSXを返す。描画中に外部変数を書き換えない（副作用は `useEffect` へ → [useeffect.md](./useeffect.md)）。

## ハマりどころ / アンチパターン
- **`class` ではなく `className`**：JSXはJSなので予約語 `class` が使えない。`for` も `htmlFor` になる。
- **JSの値は `{}` で埋め込む**：`<p>count</p>` は文字列"count"。変数を出すなら `<p>{count}</p>`。
- **小文字始まりはHTMLタグ扱い**：`<button>` は素のHTML、`<Button>` は自作コンポーネント。自作は必ず大文字始まりにする。
- **ルート要素は1つ**：複数要素を並べて返すとエラー。余計な `div` を増やしたくないなら **Fragment `<>...</>`** で包む。
- **要素を閉じ忘れる**：`<img>` や `<br>` も `<img />` `<br />` のように閉じる必要がある。
- **`if` を JSX の式の中に直接書く**：書けない。三項演算子・`&&`・早期returnを使う。`count && <X/>` は count が `0` のとき "0" が表示される罠（`count > 0 && ...` にする）。

## 関連
[props.md](./props.md)
