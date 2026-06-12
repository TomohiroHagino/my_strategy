# `view!` マクロ（Leptos）

> ⚠️ Leptos 0.x は API が変わりやすい。本書は **0.7+** 前提。細部は最新 docs で確認。

## ひとことで言うと
**HTML に近い記法で UI を書くためのマクロ**。`view! { <div>...</div> }` のように JSX 風に書け、コンパイル時に Rust のコードへ展開される。属性・イベント・動的な値をここに埋め込む。

## 役割・なぜ必要か
- Rust だけで DOM を組み立てると冗長になる。`view!` が **宣言的なテンプレート** を提供し、見た目を素直に書けるようにする。
- 重要なのは **「静的な部分」と「動く部分」を区別する** こと。`view!` は静的な骨組みを一度だけ作り、`{move || ...}` で包んだ動的な部分だけが signal の変化に追従して更新される（細粒度リアクティビティ）。→ [signals.md](./signals.md)
- イベント（クリック等）も `on:click` の形でこの中に書き、ユーザー操作を signal の更新につなぐ。

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

#[component]
fn Counter() -> impl IntoView {
    // signal() で「読み取り側」「書き込み側」のペアを作る
    let (count, set_count) = signal(0);

    view! {
        <div class="counter">
            // イベントは on:◯◯ 。クロージャで受ける
            <button on:click=move |_| set_count.update(|n| *n += 1)>
                "+1"
            </button>

            // 動的な値はクロージャで包む（これが追従して更新される部分）
            <p>"現在の値: " {move || count.get()}</p>

            // class を条件で付け外し（class:◯◯=条件クロージャ）
            <p class:active=move || count.get() > 3>
                "3より大きいと active"
            </p>

            // インラインスタイルも動的にできる
            <p style:color=move || if count.get() % 2 == 0 { "blue" } else { "red" }>
                "偶数なら青、奇数なら赤"
            </p>
        </div>
    }
}
```

```rust
// 属性は HTML と同様に書ける（静的・動的どちらも可）
view! {
    <input
        type="text"
        placeholder="名前"                  // 静的な属性
        prop:value=move || name.get()       // 動的に値をバインド（prop:value）
        on:input=move |ev| set_name.set(event_target_value(&ev))
    />
}
```

## 実務での使い方・定番パターン
- **動的な値は必ずクロージャ `{move || sig.get()}` で包む**。これが「signal が変わったらここだけ更新」の合図になる。
- **イベントは `on:click` / `on:input` / `on:submit`** など `on:` プレフィックス。`event_target_value(&ev)` で入力値を取り出すのが定番。
- **`class:active=move || ...`** で条件付きクラス、**`style:color=move || ...`** で動的スタイル。CSS フレームワークとの相性も良い。
- **`prop:value`** はフォーム要素の値を signal と双方向に近い形で結ぶときに使う（属性 `value=` と挙動が違う点に注意）。
- 条件分岐や繰り返しは `view!` に直書きせず、`Show` / `For` を使うと差分更新が効く。→ [control_flow.md](./control_flow.md)

## ハマりどころ / アンチパターン
- **動的値をクロージャで包み忘れる**：`{count.get()}` と直接書くと **最初の値で固定** され更新されない。`{move || count.get()}` が正解。
- **イベント名のプレフィックス忘れ**：`click=` ではなく `on:click=`。属性とイベントの区別を意識する。
- **クロージャ内で `count.get()` を呼ばず `count` を書く**：それは「箱」自体で値ではない。読み取りは `.get()`（または読み取り signal の参照）が必要。
- **`value=` と `prop:value=` の混同**：DOM 属性とプロパティは別物。フォームの現在値は `prop:value` 側で扱うのが安全。
- **重い計算を `{move || ...}` に直書き** すると更新のたびに走る。派生値は `Memo` にキャッシュする。→ [signals.md](./signals.md)
- 大量リスト・条件表示を生 `if` / `for` で書くと再構築が無駄に走る。専用の制御フローを使う。→ [control_flow.md](./control_flow.md)

## 関連
[signals.md](./signals.md) / [control_flow.md](./control_flow.md)
