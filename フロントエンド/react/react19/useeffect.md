# `useEffect`（副作用）（React）

## ひとことで言うと
**レンダリング（描画）が終わった後**に実行したい処理＝**副作用**を書くためのフック。データ取得・購読（subscription）・タイマー・手動でのDOM操作など、「画面を組み立てる以外のこと」をここに置く。

## 役割・なぜ必要か
- レンダリング関数は**純粋**であるべき（同じstate/propsなら同じJSXを返す、副作用なし）。なのにデータ取得やイベント購読のような副作用が必要。それを**描画後**に切り離して実行するのが `useEffect`。
- React外の世界（ブラウザAPI、サーバ、サードパーティライブラリ）と**同期**するための窓口。
- 不要になったら後始末（購読解除・タイマー停止）も必要。そのための**クリーンアップ**も同じ場所で書ける。

## 基本の書き方（コード）
```tsx
import { useState, useEffect } from "react";

function Clock() {
  const [now, setNow] = useState(() => new Date());

  useEffect(() => {
    // 描画後に実行される副作用（タイマー購読）
    const id = setInterval(() => setNow(new Date()), 1000);

    // クリーンアップ：アンマウント時／再実行前に呼ばれる
    return () => clearInterval(id);
  }, []); // 依存配列が空 = 初回マウント時のみ実行

  return <p>{now.toLocaleTimeString()}</p>;
}

function UserName({ userId }: { userId: string }) {
  const [name, setName] = useState("");

  useEffect(() => {
    let active = true; // 競合防止フラグ
    fetch(`/api/users/${userId}`)
      .then((r) => r.json())
      .then((u) => { if (active) setName(u.name); });
    return () => { active = false; }; // 古いリクエストの結果を捨てる
  }, [userId]); // userId が変わるたび再実行

  return <span>{name}</span>;
}
```

## 実務での使い方・定番パターン
- **依存配列の意味**：`[]`＝初回のみ／`[a, b]`＝`a` か `b` が変化したとき／**省略**＝毎レンダリング後（ほぼ使わない）。
- **クリーンアップ**：`return () => {...}` で購読解除・`clearInterval`・`AbortController.abort()` を行う。次のeffect実行前とアンマウント時に呼ばれる。
- **競合（race condition）対策**：非同期取得は `active` フラグや `AbortController` で、古いレスポンスを捨てる。
- **多くの場合、自前のデータ取得effectは不要**：実務では TanStack Query / SWR に任せるのが定番（キャッシュ・再取得・競合処理込み）。→ [data_fetching.md](./data_fetching.md)
- **イベント購読**：`window.addEventListener` 系はeffectで登録、クリーンアップで解除する。

## ハマりどころ / アンチパターン
- **依存配列の漏れ（stale closure）**：effect内で使う state/props/関数を依存に入れ忘れると、**古い値を掴んだまま**動く。`eslint-plugin-react-hooks` の `exhaustive-deps` 警告に従う。→ [hooks.md](./hooks.md)
- **無限ループ**：effect内で依存しているstateを更新する、または**オブジェクト/配列/関数を依存に直接置く**（毎回新しい参照になる）と再実行が止まらない。`useMemo`/`useCallback` で安定化、または依存を見直す。→ [performance.md](./performance.md)
- **クリーンアップ忘れ**：タイマー・購読を解除しないとメモリリークや「外れたコンポーネントへのsetState」警告に。
- **StrictModeで開発時に二重実行**：React 18+ の開発モードはeffectを意図的に2回実行し、クリーンアップ漏れを炙り出す。本番では1回。**クリーンアップを正しく書けば問題ない**設計を促す挙動。
- **本来不要なeffect**：「propsから派生する値の計算」「イベントへの応答」はeffectでなく、レンダリング中の計算やイベントハンドラで。→ [state.md](./state.md)

## 関連
[hooks.md](./hooks.md) / [data_fetching.md](./data_fetching.md) / [state.md](./state.md)
