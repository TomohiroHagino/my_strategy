# Effect / Memo（副作用・派生値）（Leptos 0.7+）

## ひとことで言うと
- **`Effect`**：signal が変わったときに走らせたい**副作用**（ログ出力・外部 API 同期・`localStorage` 書き込みなど、UI 描画以外の処理）。
- **`Memo`**：他の signal から計算した**派生値**を**キャッシュ**し、入力が変わったときだけ再計算する箱。

## 役割・なぜ必要か
- **Effect** は「リアクティブな世界」と「外の世界（DOM 直接操作・ブラウザ API・ネットワーク）」をつなぐ出口。中で読んだ signal を**自動で購読**し、変わるたびに再実行される。
- **Memo** は「重い派生計算を毎回やりたくない」「複数箇所で同じ派生値を使う」ときに効く。値が実際に変わったときだけ下流へ通知するので、無駄な再評価を抑える。
- 単純な派生はわざわざ Memo にせず、view 内クロージャでも十分（後述）。

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

#[component]
fn Demo() -> impl IntoView {
    let (count, set_count) = signal(0);

    // Effect: 中で使った signal が変わるたび再実行（副作用専用）
    Effect::new(move |_| {
        // count.get() を読むので count を購読する
        leptos::logging::log!("count changed: {}", count.get());
    });

    // Memo: 入力(count)が変わったときだけ再計算し、結果をキャッシュ
    let doubled = Memo::new(move |_| count.get() * 2);

    view! {
        <p>"x2 = " {move || doubled.get()}</p>
        <button on:click=move |_| set_count.update(|n| *n += 1)>"+1"</button>
    }
}
```

```rust
// 単なる派生（軽い計算・キャッシュ不要）はクロージャでよい
let doubled = move || count.get() * 2;
view! { <p>{doubled}</p> }
```

`Effect::new` のクロージャ引数 `_` は**前回の戻り値**（初回は `None`）。
差分を取りたいときに使えるが、使わなければ `|_|` で無視してよい。

## 実務での使い方・定番パターン
- **Effect は「外部同期」に限定**：テーマを `localStorage` に保存、`document.title` 更新、サードパーティ JS へ値を渡す、など。**UI は view で表現**し、Effect で DOM を書き換えない。
- **Memo は重い派生 / 多くの購読者がいる派生に**：フィルタ済みリスト、整形済み文字列、合計値など。値が同じなら下流を再実行しないので、再描画も抑えられる。
- **Memo は `PartialEq` で変化判定**：結果が前回と等しければ通知しない。だから「同じ値での無駄な更新」を自然に防げる。
- **初期化系の副作用**：マウント時に一度だけ走らせたい処理も Effect に置ける（クライアント側で実行される）。
- **Effect の中で signal を書くのは慎重に**：書いた先を同じ Effect が購読していると再実行ループになる。書くなら購読していない signal に。

## ハマりどころ / アンチパターン
- **Effect で UI を作る**：DOM を直接いじらない。表示は **view** とクロージャ購読で行う。Effect は副作用専用。
- **派生値を Effect で signal に書き戻す**：`Effect で計算 → set_signal` は典型的アンチパターン。**Memo（やクロージャ）で派生**させる方が正しく、ループも起きない。
- **旧 API を使う**：`create_effect` / `create_memo` は 0.7 系では**非推奨/廃止**方向。**`Effect::new` / `Memo::new`** を使う。
- **何でも Memo 化**：軽い派生まで Memo にすると逆に冗長。`move || ...` クロージャで足りるなら Memo は不要（→ [signals.md](./signals.md)）。
- **Effect で `get_untracked` 多用**：購読されず再実行されなくなり「動かない Effect」に。購読すべき値は普通に `get()` で読む。
- **SSR での Effect 期待**：Effect は基本クライアント実行。サーバ側描画時には走らない前提で設計する（→ ssr_csr.md）。

## 関連
[signals.md](./signals.md) / [control_flow.md](./control_flow.md)
