# 再レンダリングと最適化（memo / useMemo / useCallback）（React）

## ひとことで言うと
**親が再レンダリングされると子も既定で再レンダリングされる**。それが重いときに `React.memo`(子をスキップ)・`useMemo`(計算をメモ化)・`useCallback`(関数参照を固定)で**ムダな再計算・再描画を減らす**仕組み。

## 役割・なぜ必要か
- Reactの再レンダリングは「state/propsが変わる → そのコンポーネントと**子孫を再実行**して仮想DOMを作り直す」。多くは速いので**普段は気にしなくていい**。
- ただしリストが巨大・計算が重い・子が多い場合に体感が落ちる。そのとき**該当箇所だけ**メモ化で抑える。
- 大前提：**まず計測してから最適化**する。React DevTools の Profiler で「どこが・なぜ再描画したか」を見てから手を入れる。あてずっぽうの memo は逆効果。

## 基本の書き方（コード）
```tsx
import { memo, useMemo, useCallback, useState } from "react";

// React.memo: propsが前回と同じなら子の再レンダリングをスキップ
const HeavyChild = memo(function HeavyChild(props: {
  label: string;
  onClick: () => void;
}) {
  console.log("child rendered"); // propsが変わった時だけ出る
  return <button onClick={props.onClick}>{props.label}</button>;
});

export function Parent() {
  const [count, setCount] = useState(0);
  const [items] = useState([3, 1, 2]);

  // useMemo: itemsが変わらない限り、重い並べ替えを再計算しない
  const sorted = useMemo(() => {
    return [...items].sort((a, b) => a - b);
  }, [items]); // 依存配列。itemsが同じなら前回の結果を使い回す

  // useCallback: countが変わっても関数の「参照」を固定 → memoが効く
  const handleClick = useCallback(() => {
    console.log("clicked");
  }, []); // 依存なし＝常に同じ関数インスタンス

  return (
    <div>
      <p>count: {count} / sorted: {sorted.join(",")}</p>
      <button onClick={() => setCount((c) => c + 1)}>+1</button>
      <HeavyChild label="子ボタン" onClick={handleClick} />
    </div>
  );
}
```

## 実務での使い方・定番パターン
- **`React.memo`**：propsが浅く等しい(`===`)なら再レンダリングを飛ばす。再描画コストの高い・propsが安定した子に付ける。
- **`useMemo`**：重い計算(ソート・フィルタ・集計)の結果をキャッシュ。依存配列が変わらない限り前回値を返す。**軽い計算には不要**(管理コストの方が高い)。
- **`useCallback`**：関数を毎レンダリング作り直すと参照が変わり、`memo`した子に「新しいprop」と誤認される。`useCallback`で参照を固定すると memo が効く。**memoした子へ関数を渡すとき**が主用途。
- セット運用が定番：`memo`した子へ `useCallback`の関数 と `useMemo`のオブジェクト/配列を渡す。3つは**併用して初めて効く**ことが多い。
- 計測は Profiler の "Why did this render?" を確認。Context の値も `useMemo` で包むと不要な再描画を抑えられる。→ [context.md](./context.md)

## ハマりどころ / アンチパターン
- **早すぎる最適化**：効果のない場所に memo/useMemo を撒くと、依存比較のコストとコードの複雑さだけ増える。**計測で遅いと分かってから**。
- **依存配列ミス**：`useMemo`/`useCallback` の依存に必要な値を入れ忘れると、**古い値を掴んだまま**(stale)になりバグる。eslint の `react-hooks/exhaustive-deps` を有効に。
- **`memo`の誤解**：親が**毎回新しいオブジェクト/配列/関数をprops で渡す**と、参照が毎回変わり memo は**まったく効かない**。渡す値側も `useMemo`/`useCallback` で安定させる必要がある。
- インラインの `style={{...}}` や `onClick={() => ...}` も毎回新規生成。memoした子に渡すなら要注意。
- React 19 の **React Compiler** が自動でメモ化を入れる方向。手動 memo の必要性は将来減る見込み。

## 関連
[hooks.md](./hooks.md) / [context.md](./context.md)
