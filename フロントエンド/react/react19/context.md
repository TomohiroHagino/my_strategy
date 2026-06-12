# `useContext`（プロップドリリング回避）（React）

## ひとことで言うと
`createContext` で「共有の入れ物」を作り、`Provider` で値を流し込み、子孫が `useContext` で**props を経由せず直接受け取る**仕組み。テーマ・ログインユーザ・言語のような「アプリ全体で使い回す値」を配るのに使う。

## 役割・なぜ必要か
- props は親→子へ手渡しなので、深いツリーでは**通り道の中間コンポーネントが使わない props を延々と中継**することになる（＝プロップドリリング）。Context はこの中継をスキップする。
- 「多くの場所が同じ値を読む」ものに向く（現在のテーマ、ログイン状態、ロケール、`react-router` の現在地など）。
- 配るのは**値そのもの**だけでなく、**更新関数**もまとめて渡せる（`{ user, login, logout }`）。状態管理ライブラリを入れる前の素朴な共有手段。

## 基本の書き方（コード）
```tsx
import { createContext, useContext, useState, type ReactNode } from "react";

// 1) 型を決めてContextを作る（既定値はnullにして未提供を検知）
type Theme = "light" | "dark";
type ThemeContextValue = {
  theme: Theme;
  toggle: () => void;
};
const ThemeContext = createContext<ThemeContextValue | null>(null);

// 2) Provider用のラッパーを作る（値の生成はここに集約）
export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>("light");
  const toggle = () => setTheme((t) => (t === "light" ? "dark" : "light"));
  return (
    <ThemeContext.Provider value={{ theme, toggle }}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3) 専用フックでnullチェックを1か所に閉じ込める
export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext);
  if (ctx === null) {
    throw new Error("useTheme は <ThemeProvider> の内側で使ってください");
  }
  return ctx;
}

// 4) 消費側はpropsを一切受け取らずに値を読める
function ThemeButton() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>現在: {theme}</button>;
}

// 5) ツリーの上位でProviderで包む
function App() {
  return (
    <ThemeProvider>
      <ThemeButton />
    </ThemeProvider>
  );
}
```

## 実務での使い方・定番パターン
- **「Context＋専用フック」をセットで公開**する。`useContext` を直接呼ばせず `useTheme()` 越しにし、未提供時のエラーと型の `null` 除去を一箇所にまとめる。→ [hooks.md](./hooks.md)
- **Provider は専用コンポーネントに切り出す**（上の `ThemeProvider`）。値の生成ロジックを Provider 内に閉じ込め、`App` 側はただ包むだけにすると見通しが良い。
- **更新頻度で Context を分ける**。「ほぼ変わらない値（テーマ・ロケール）」と「よく変わる値」を同じ Context に混ぜない。混ぜると後者の更新で前者の消費者まで巻き込む。
- 認証では `AuthProvider` が定番。`{ user, login, logout, isLoading }` を配り、ルーティングのガードから読む。→ [routing.md](./routing.md)
- 複数 Provider が重なるなら、まとめた `AppProviders` を1つ作って `App` をスッキリ保つ。

## ハマりどころ / アンチパターン
- **値が変わると、その Context を読む全消費者が再レンダリングされる**。これが最大の落とし穴。緩和策は ①更新頻度ごとに Context を分割、②`value` をむやみに新オブジェクトで作り直さない（`useMemo` で安定化）、③重い子は `React.memo` で隔離。→ [performance.md](./performance.md)
- **`value={{ ... }}` を毎レンダリングで新規生成**すると、中身が同じでも参照が変わり再レンダリングが走る。`useMemo(() => ({ theme, toggle }), [theme])` のように包む。
- **何でも Context に入れない**。とくに**サーバ状態（API取得データ）は Context ではなく TanStack Query / SWR へ**。Context はキャッシュ・再取得・stale 管理をしてくれない。→ [data_fetching.md](./data_fetching.md)
- **既定値に本物の値を置かない**。`createContext<T | null>(null)` にして「Provider 外で使った」事故を検知できるようにする。`createContext({} as T)` は未提供バグを黙って通す。
- **Provider の外で消費**すると既定値（や `null`）が返り、気づきにくい。専用フックの `throw` で早期に弾く。
- グローバル state の万能薬ではない。書き換えが頻繁・粒度が細かいなら Zustand 等を検討。→ [props.md](./props.md)

## 関連: [props.md](./props.md) / [performance.md](./performance.md) / [data_fetching.md](./data_fetching.md)
