# フックのルールとカスタムフック（React）

## ひとことで言うと
**フック（Hook）** は `use` で始まる関数（`useState` / `useEffect` など）で、関数コンポーネントに state やライフサイクルなどの機能を「接続」する仕組み。安全に動かすための**呼び出しルール**があり、自作してロジックを再利用する**カスタムフック**も作れる。

## 役割・なぜ必要か
- React はフックを**呼ばれた順番**で各stateと結びつけて管理している。だから「毎回同じ順序・同じ回数」呼ばれることが大前提。これが**フックのルール**の理由。
- コンポーネント間で**ロジック（state＋effectのまとまり）を再利用**したいとき、HOCやrender propsより素直に書けるのがカスタムフック。
- ルールを守ることで、Reactが内部状態を取り違えずに済み、バグを防げる。

## 基本の書き方（コード）
```tsx
import { useState, useEffect } from "react";

// === フックのルール ===
// 1. トップレベルでのみ呼ぶ（条件・ループ・ネスト関数の中はNG）
// 2. 呼べるのは「Reactコンポーネント」か「カスタムフック」の中だけ
function Bad({ on }: { on: boolean }) {
  if (on) {
    // NG: 条件分岐の中。呼ばれる回数が変わり順序が崩れる
    const [v] = useState(0);
  }
  return null;
}

// === カスタムフック（use で始める・ロジックを再利用） ===
function useWindowWidth(): number {
  const [width, setWidth] = useState(() => window.innerWidth);
  useEffect(() => {
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize);
  }, []);
  return width; // state でも値でも自由に返せる
}

function Banner() {
  const width = useWindowWidth(); // 中の state/effect は呼び出し元ごとに独立
  return <p>{width < 768 ? "モバイル" : "PC"}</p>;
}
```

## 実務での使い方・定番パターン
- **カスタムフックは `use` 始まり**にする（lintルール・順序検出のため必須の命名）。中で他のフックを呼べる。
- **抽出の単位**：「同じstate＋effectの組み合わせ」が2箇所以上に出たら、カスタムフックへ切り出す（`useToggle`, `useDebounce`, `useLocalStorage` など）。
- **stateは共有されない**：同じカスタムフックを複数コンポーネントで使っても、stateは**それぞれ独立**。共有したいなら Context や状態管理ライブラリを使う。→ [context.md](./context.md)
- **`useMemo` / `useCallback`（概要）**：`useMemo(fn, deps)` は計算結果を、`useCallback(fn, deps)` は関数を**メモ化**して、依存が変わらない限り同じ値/参照を返す。主目的は**不要な再計算・再生成・子の再レンダリング抑制**。乱用は逆効果なので、計測してから使う。詳細→ [performance.md](./performance.md)
- **eslint-plugin-react-hooks を必ず有効化**：ルール違反と依存漏れを自動検出してくれる。

## ハマりどころ / アンチパターン
- **条件付き・ループ内での呼び出し**：`if (cond) useX()` や `for` 内 `useY()` は**順序が崩れて壊れる**。条件は早期returnの「前」にフックをすべて呼んでから。
- **早期returnの後にフック**：`if (!data) return null;` の**後**に `useState` を置くとレンダリングによって呼ばれたり呼ばれなかったりしてNG。フックは関数の先頭にまとめる。
- **通常関数からフックを呼ぶ**：コンポーネントでもカスタムフックでもない普通の関数内で `useState` などを呼ぶのは違反。
- **`useMemo`/`useCallback` の貼りすぎ**：メモ化自体にコストがあり、依存配列の管理も増える。効果が出る場所（重い計算・`React.memo` 化した子への参照安定）に限定する。→ [performance.md](./performance.md)
- **依存漏れ**：`useCallback`/`useMemo`/`useEffect` の依存配列の漏れは stale な値の温床。`exhaustive-deps` に従う。→ [useeffect.md](./useeffect.md)

## 関連
[performance.md](./performance.md) / [useeffect.md](./useeffect.md) / [state.md](./state.md)
