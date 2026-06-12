# `useRef` / DOM参照（React）

## ひとことで言うと
`useRef` は **`.current` に何かを入れておける箱**を返すフック。用途は2つ。① **DOM要素を直接つかむ**（`ref={ref}` → `ref.current` がその要素）、② **再レンダリングを起こさずに値を持ち回る**（タイマーIDや前回値など）。

## 役割・なぜ必要か
- React は基本「状態 → 画面」で DOM を直接触らない設計だが、**フォーカス・スクロール・テキスト選択・`<video>` 再生・外部DOMライブラリ連携**など、どうしても生の DOM が要る場面がある。そこで `ref` を使う。
- `useState` は値が変わると再レンダリングするが、`useRef` の `.current` は**書き換えても再レンダリングしない**。だから「画面に出さないが覚えておきたい値（`setInterval` のID、直前のprops、初回判定フラグ）」の保管に向く。
- レンダリングをまたいで**同じ箱を保持し続ける**（毎回作り直されない）のが `useState` と共通の特徴。

## 基本の書き方（コード）
```tsx
import { useRef, useEffect, useState, forwardRef } from "react";

// 用途① DOM参照：マウント直後に入力欄へフォーカス
function SearchBox() {
  const inputRef = useRef<HTMLInputElement>(null); // 初期はnull
  useEffect(() => {
    inputRef.current?.focus(); // currentがnullの可能性に注意（?. で守る）
  }, []);
  return <input ref={inputRef} placeholder="検索" />;
}

// 用途② 再レンダリングを起こさない可変値：タイマーIDの保持
function Timer() {
  const [count, setCount] = useState(0);
  const idRef = useRef<number | null>(null); // 画面に出さない値はrefでOK
  const start = () => {
    if (idRef.current !== null) return;
    idRef.current = window.setInterval(() => setCount((c) => c + 1), 1000);
  };
  const stop = () => {
    if (idRef.current !== null) {
      clearInterval(idRef.current);
      idRef.current = null;
    }
  };
  return (
    <>
      <p>{count}秒</p>
      <button onClick={start}>開始</button>
      <button onClick={stop}>停止</button>
    </>
  );
}

// forwardRef：子の内部DOMを親から操作できるようにする（React 18/19）
const FancyInput = forwardRef<HTMLInputElement, { label: string }>(
  function FancyInput({ label }, ref) {
    return (
      <label>
        {label}
        <input ref={ref} /> {/* 親が渡したrefを内部の<input>へ転送 */}
      </label>
    );
  }
);

function Parent() {
  const ref = useRef<HTMLInputElement>(null);
  return (
    <>
      <FancyInput label="名前" ref={ref} />
      <button onClick={() => ref.current?.focus()}>フォーカス</button>
    </>
  );
}
```

## 実務での使い方・定番パターン
- **DOM参照**：フォーカス制御、`scrollIntoView`、`<canvas>`/`<video>` 操作、地図やチャートなど非React製ライブラリのマウント先として。
- **再レンダリング不要の可変値**：`setInterval`/`setTimeout` のID、`AbortController`、`requestAnimationFrame` のハンドル、WebSocket インスタンスなどの保管。→ [useeffect.md](./useeffect.md)
- **「前回の値」を覚える**：`useEffect` 内で `prevRef.current = value` と更新し、次回レンダリングで前回値と比較する小技。
- **`forwardRef`** で再利用 UI 部品（入力・モーダル）の内部DOMを親へ公開。React 19 では `ref` を通常 prop として受け取れる方向に簡素化されている点も押さえておく。
- 子のメソッドだけ公開したいときは `useImperativeHandle` と併用（多用は避ける）。

## `useMemo` との違い（混同しやすい）
どちらも「レンダーをまたいで値を保持する」ので紛らわしいが、**目的は真逆**。

| | `useRef` | `useMemo` |
|---|---|---|
| 何をする | 値を入れる**「箱」**を保持（`{ current }`） | **計算結果**をキャッシュ |
| 中身の決まり方 | **自分で代入**（`ref.current = x`） | **関数の戻り値**（依存から計算） |
| 再計算 | しない（ずっと同じ箱） | **依存配列が変わった時だけ**再計算 |
| 変更したら | **再レンダリングしない** | （依存変化で）レンダー時に新しい値になる |
| 主な用途 | DOM参照 / 再描画したくない可変値 | 重い計算の節約 / 参照(`===`)の安定化 |

```tsx
// useRef：再描画を起こさず値を保持する「箱」。中身は自分で代入
const count = useRef(0);
count.current++;                               // 何度やっても再レンダリングなし

// useMemo：依存が同じ間は計算をスキップしてキャッシュ。中身は計算結果
const sorted = useMemo(() => heavySort(list), [list]); // listが変わった時だけ再計算
```

### 「一度だけ作りたい」は useRef が正解
`useMemo(() => new Heavy(), [])` で初回生成っぽく書けるが、**useMemo はキャッシュが破棄され得る**（Reactの仕様上、保持は保証されない）。確実に1回だけなら ref を使う:
```tsx
const ref = useRef<Heavy>();
if (!ref.current) ref.current = new Heavy();   // 初回だけ生成、以後同じインスタンス
```
→ 最適化（計算キャッシュ）が目的なら useMemo、保持が目的なら useRef。詳しくは [performance.md](./performance.md)。

## ハマりどころ / アンチパターン
- **`ref` の値はレンダリング結果に使わない**。`.current` を書き換えても**再レンダリングは起きない**ので、画面に映したい値は `useState` を使う。`{ref.current}` を JSX に直書きすると古い値が残る。
- **初期 `null` の扱い**：DOM ref は初回レンダリング〜マウント前は `null`。`ref.current.focus()` ではなく `ref.current?.focus()` と必ずガードする。
- **レンダリング中に `.current` を読み書きしない**。副作用は `useEffect`/イベントハンドラの中で。レンダリング中の ref 書き換えは予期せぬ挙動の元。
- **DOM操作は最小限に**。クラス付与・表示/非表示・要素の出し入れは**状態でやる**のが基本。ref で直接 DOM をいじると React の管理と二重管理になり破綻する。→ [state.md](./state.md)
- **`ref` を依存配列に入れない**：ref オブジェクトは安定（同一参照）なので effect の依存に書いても意味がない。
- タイマー/購読を ref に持ったら、**`useEffect` のクリーンアップで必ず解除**してリーク防止。

## 関連: [useeffect.md](./useeffect.md) / [state.md](./state.md) / [performance.md](./performance.md)（useMemo/useCallback）
