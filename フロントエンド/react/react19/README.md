# React 実務リファレンス（索引）

> **対象 = React 18/19（TypeScript前提）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> React は「**UIを部品（コンポーネント）に分け、状態(state)から画面を組み立てる**」ライブラリ。フレームワークではないので、ルーティングやデータ取得は別ライブラリを足す。

## 核となる考え方
```
 状態(state) が変わる → React が再レンダリング → 画面が状態に追従
 UI = f(state)（画面は状態の関数）。手でDOMをいじらず、状態を変える。
```

## 項目（各ファイルへ）

### はじめに / 基礎
- [getting_started.md](./getting_started.md) … 始め方（Vite / コンポーネント / 起動）
- [components_jsx.md](./components_jsx.md) … コンポーネントと JSX とは
- [props.md](./props.md) … props（親→子のデータ）とは
- [state.md](./state.md) … state と `useState` とは
- [lists_keys.md](./lists_keys.md) … リスト描画と key とは
- [forms.md](./forms.md) … フォーム（制御コンポーネント）とは

### フック / 仕組み
- [useeffect.md](./useeffect.md) … `useEffect`（副作用）とは
- [hooks.md](./hooks.md) … フックのルールとカスタムフックとは
- [context.md](./context.md) … `useContext`（プロップドリリング回避）とは
- [redux.md](./redux.md) … Redux（アプリ全体の状態管理 / Redux Toolkit）とは
- [refs.md](./refs.md) … `useRef` / DOM参照とは
- [performance.md](./performance.md) … 再レンダリングと最適化（memo / useMemo / useCallback）とは

### 周辺・運用
- [request_flow.md](./request_flow.md) … データの流れ（状態→UIの一方向＋データ取得フロー）・各部分が何を返すか
- [routing.md](./routing.md) … ルーティング（React Router）とは
- [data_fetching.md](./data_fetching.md) … データ取得（TanStack Query / SWR）とは
- [testing.md](./testing.md) … テスト（Testing Library）とは
- [testing_library.md](./testing_library.md) … React Testing Library（render / getByRole / userEvent）とは
- [msw.md](./msw.md) … MSW（ネットワーク層でAPIモック）とは
- [playwright.md](./playwright.md) … Playwright（実ブラウザE2E）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

> 関連: Reactの上に乗るフルスタック版は [../nextjs/](../../nextjs/nextjs15/)（SSR・ルーティング等を内蔵）。

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（React）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
