# Redux（React）

## ひとことで言うと
アプリ全体で共有する**クライアント状態を1か所（store）にまとめて管理**するライブラリ。**単方向データフロー**（状態は store にだけあり、`action` を `dispatch` して `reducer` が新しい状態を返す）で、状態変化を予測可能にする。現代は公式の **Redux Toolkit（RTK）** を使うのが標準。

## 役割・なぜ必要か
- `useState` は1コンポーネントの状態、Context は「変化が少ない共有値」向き。**アプリ全体にまたがる・頻繁に変わる・複雑な状態**（カート、認証、複数画面で共有する編集中データ等）を、秩序立てて扱うために使う。
- **ただし多くの場合は不要**。「とりあえずRedux」は過剰になりがちで、まず `useState`/Context/軽量な Zustand、**サーバ由来データは TanStack Query** で足りないかを先に考える。→ [data_fetching.md](./data_fetching.md)

## 基本の書き方（コード）— Redux Toolkit
```ts
// store/counterSlice.ts
import { createSlice } from "@reduxjs/toolkit";

const counterSlice = createSlice({
  name: "counter",
  initialState: { value: 0 },
  reducers: {
    // RTKは内部でimmerを使うので「直接書き換える」書き方でOK（実際は新state）
    increment: (state) => { state.value += 1; },
    addBy: (state, action: { payload: number }) => { state.value += action.payload; },
  },
});
export const { increment, addBy } = counterSlice.actions;
export default counterSlice.reducer;
```
```ts
// store/index.ts
import { configureStore } from "@reduxjs/toolkit";
import counter from "./counterSlice";
export const store = configureStore({ reducer: { counter } });
export type RootState = ReturnType<typeof store.getState>;
```
```tsx
// main.tsx：Providerで包む
<Provider store={store}><App /></Provider>

// コンポーネント：読み取りと更新
const value = useSelector((s: RootState) => s.counter.value); // 読む
const dispatch = useDispatch();
<button onClick={() => dispatch(increment())}>+1</button>      // 変える
```

## 実務での使い方・定番パターン
- **`createSlice` で「状態＋更新ロジック＋action」をまとめて定義**（旧来の action types / action creators の定型文が消える）。
- **非同期**：`createAsyncThunk`（API呼び出し→pending/fulfilled/rejected）。
- **サーバデータの取得・キャッシュは RTK Query**（または TanStack Query）。Reduxの手書きより楽。
- **状態の置き場所を分ける**：UI/クライアント状態＝Redux、**サーバ状態＝Query系**、URL状態＝ルーター。混ぜない。
- 小〜中規模なら Redux を入れず **Zustand** 等の軽量ストアも有力。

## ハマりどころ / アンチパターン
- **何でも Redux に入れる**：1コンポーネントで足りる状態まで store に上げると複雑化。まず `useState`。
- **サーバ状態を Redux で持つ**：再取得・キャッシュ・無効化を自前で書く羽目に。→ Query系に任せる（[data_fetching.md](./data_fetching.md)）。
- **`useSelector` で毎回新しいオブジェクト/配列を返す**：参照が変わり毎回再レンダリング。必要な値だけ選ぶ／`createSelector` でメモ化。
- **古い「素の Redux」の定型文**：今は **Redux Toolkit が公式推奨**。`createStore` 手書きや大量の boilerplate は避ける。
- **immerの誤解**：`createSlice` 内は直接書き換え風でOKだが、それ以外（セレクタ等）で state を変更しない。

## 関連
[state.md](./state.md) / [context.md](./context.md) / [data_fetching.md](./data_fetching.md)
