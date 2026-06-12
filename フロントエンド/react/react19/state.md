# state と `useState`（React）

## ひとことで言うと
コンポーネントが**自分で持つ、変化する値**。`useState` フックで宣言し、`state` が変わると React がそのコンポーネントを**再レンダリング**して画面を更新する。`UI = f(state)` の `state` の部分。

## 役割・なぜ必要か
- 「入力中の文字」「開閉フラグ」「取得したデータ」など、**時間とともに変わる値**を保持するため。ただのローカル変数だと再代入しても再レンダリングが起きず、画面に反映されない。
- React の基本方針は **DOMを手でいじらず、状態を変える**。状態を変えれば画面が状態に追従する（宣言的UI）。
- これにより「今のDOMの状態」を気にせず、「あるべき画面 = 状態の関数」だけを書けばよくなる。

## 基本の書き方（コード）
```tsx
import { useState } from "react";

function Counter() {
  // [現在値, 更新関数] = useState(初期値)
  const [count, setCount] = useState<number>(0);

  return (
    <div>
      <p>カウント: {count}</p>
      {/* setCount を呼ぶと再レンダリングされ、count が新しい値になる */}
      <button onClick={() => setCount(count + 1)}>+1</button>
    </div>
  );
}

// オブジェクト/配列は「新しいもの」を作って渡す（immutable）
function Profile() {
  const [user, setUser] = useState({ name: "太郎", age: 20 });
  const [tags, setTags] = useState<string[]>([]);

  const birthday = () => {
    // NG: user.age++  /  OK: 新しいオブジェクトを作る（スプレッド）
    setUser({ ...user, age: user.age + 1 });
  };
  const addTag = (t: string) => {
    // NG: tags.push(t)  /  OK: 新しい配列を作る
    setTags([...tags, t]);
  };

  return <button onClick={birthday}>{user.name}: {user.age}歳</button>;
}
```

## 実務での使い方・定番パターン
- **関数型更新**：直前の値を元に更新するなら `setCount(prev => prev + 1)`。連続更新やバッチでも安全（古い `count` を参照しない）。
- **immutable更新を徹底**：オブジェクトは `{ ...obj, key: v }`、配列は `[...arr, x]` / `arr.filter(...)` / `arr.map(...)` で**新しい参照**を作る。React は参照の変化で再描画を判断するため、これが必須。
- **初期値が重い計算なら遅延初期化**：`useState(() => expensive())` と関数を渡すと初回だけ実行される。
- **state vs props**：`state` は「自分で持ち、自分で変える」値。`props` は「親から受け取る、読み取り専用」の値（子は書き換えない）。→ [props.md](./props.md)
- **stateは最小限に**：「他のstateやpropsから計算できる値」は持たず、レンダリング中に**派生**させる（例：`const fullName = first + last`）。

## ハマりどころ / アンチパターン
- **直接変更（mutation）**：`arr.push(x)` や `obj.age = 1` してから `setX(arr)` を渡しても、**同じ参照**なので再描画されない／されても挙動が不安定。必ず新しいオブジェクト/配列を渡す。
- **stateは即時反映ではない**：`setCount(count+1)` の直後に `count` を読んでも**まだ古い**。Reactは更新をまとめて適用（バッチ）し、次のレンダリングで新しい値になる。連続更新は関数型更新で。
- **派生値をstateにする**：`fullName` を別stateで持つと、元stateと**ズレる**（同期漏れ）。計算で導く。→ [useeffect.md](./useeffect.md)（無駄なeffectでの同期も避ける）
- **`const [x] = useState(props.value)`**：これは**初期値**にしかならない。props変更を追従させたいなら別の設計（keyで作り直す等）を検討。

## 関連
[props.md](./props.md) / [useeffect.md](./useeffect.md) / [hooks.md](./hooks.md)
