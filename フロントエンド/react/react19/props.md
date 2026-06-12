# props（親→子のデータ）（React）

## ひとことで言うと
**親コンポーネントから子コンポーネントへ渡すデータ**。HTMLの属性のように `<Child name="..." />` と書いて渡す。子から見ると**読み取り専用**の入力値。

## 役割・なぜ必要か
- コンポーネントを「同じ見た目・違う中身」で**再利用**するための仕組み。`<Button label="保存" />` と `<Button label="削除" />` のように、1つの部品を値違いで使い回せる。
- **UI = f(props, state)**：propsは関数コンポーネントの「引数」にあたる。親が渡す値が変われば子は再レンダリングされ、画面が追従する。
- データは**上から下へ（親→子）一方向**に流れる。流れが一方向なので、どこで値が決まり何が表示されるか追いやすい。

## 基本の書き方（コード）
TypeScriptで型を定義し、**分割代入**で受け取る:
```tsx
// props の型を定義（PascalCase で Props と名付けるのが定番）
type GreetingProps = {
  name: string;
  count?: number;             // ? は省略可能（任意）
  children?: React.ReactNode; // タグの中身を受け取る特別な prop
};

// 分割代入で受ける。count は既定値を与えられる
function Greeting({ name, count = 0, children }: GreetingProps) {
  return (
    <section>
      <h2>こんにちは、{name} さん</h2>
      <p>通知 {count} 件</p>
      {children} {/* 親がタグの中に書いた要素がここに入る */}
    </section>
  );
}

// 親から渡す側
function App() {
  return (
    <Greeting name="田中" count={3}>
      <button>詳細を見る</button> {/* これが children として渡る */}
    </Greeting>
  );
}
```

## 実務での使い方・定番パターン
- **型は `type Props = {...}`** で先頭に定義。boolean は `isOpen` / `hasError`、コールバックは `onSelect` のように命名。
- **`children`** でレイアウト部品を作る（`<Card>中身</Card>`）。中身を差し替えられる汎用部品になる。
- イベントは「**データは下へ、通知は上へ**」：親が関数を props で渡し、子はそれを呼ぶ（`<Child onSubmit={handleSubmit} />`）。子が直接親の状態を触らない。
- propsが多すぎる/深い階層を貫通させる（**プロップドリリング**）なら、**Context** でまとめて配る（→ [context.md](./context.md)）。

## ハマりどころ / アンチパターン
- **propsは変更不可（読み取り専用）**：子で `props.name = "x"` のように書き換えてはいけない。値を変えたいのは親側の **state**（→ [state.md](./state.md)）。
- **プロップドリリング**：使わない中間コンポーネントを、ただ下へ渡すためだけに props がバケツリレーで貫通する状態。階層が深いと修正が地獄。→ Context で解決。
- `{}` の付け方：文字列はそのまま `name="田中"`、数値・真偽・式は `count={3}` `disabled={isLoading}` と `{}` で囲う。
- **オブジェクトや配列を props で毎回新規生成**して渡すと、子の `React.memo` が効かず再レンダリングが増える。→ [performance.md](./performance.md)
- 「任意」props（`count?`）は `undefined` で来うる。既定値（`count = 0`）か `count ?? 0` でガードする。
- スプレッド（`<Comp {...props} />`）の渡しすぎは「何が渡っているか」が不透明になり追いにくい。必要な props を明示する。

## 関連
[state.md](./state.md) / [context.md](./context.md)
