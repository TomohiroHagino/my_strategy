# 制御フロー（Show / For）（Leptos 0.7+）

## ひとことで言うと
view の中で「**条件で出し分ける**」「**リストを繰り返す**」を、細粒度リアクティビティに乗せて行うためのコンポーネント。
- **`<Show>`**：条件付き描画（`when` が真なら子、偽なら `fallback`）。
- **`<For>`**：リスト描画。各要素を一意に識別する **`key` が必須**。

## 役割・なぜ必要か
- **`if` / `for` を view にそのまま書けない**ため、リアクティブに追従する専用コンポーネントを使う。条件や配列が signal 由来で変わったとき、必要な部分だけ更新される。
- **`Show`** は「条件が真のときだけ描く」を効率よく行う（条件が変わったときだけ切り替わる）。
- **`For`** は `key` で要素を同定し、**追加・削除・並べ替えを最小差分で**反映する。key がないと毎回作り直しになり、状態（入力フォーカス等）も失われやすい。

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

#[component]
fn Toggle() -> impl IntoView {
    let (open, set_open) = signal(false);

    view! {
        <button on:click=move |_| set_open.update(|o| *o = !*o)>"切替"</button>

        // when / fallback はクロージャで渡す
        <Show
            when=move || open.get()
            fallback=|| view! { <p>"閉じています"</p> }
        >
            <p>"開いています"</p>
        </Show>
    }
}
```

```rust
#[derive(Clone)]
struct Item { id: usize, label: String }

#[component]
fn List() -> impl IntoView {
    let (items, _set_items) = signal(vec![
        Item { id: 1, label: "あ".into() },
        Item { id: 2, label: "い".into() },
    ]);

    view! {
        <ul>
            <For
                each=move || items.get()      // 反復対象（クロージャ）
                key=|item| item.id            // 一意キー（必須）
                let:item                       // 各要素を item で受ける
            >
                <li>{item.label.clone()}</li>
            </For>
        </ul>
    }
}
```

## 実務での使い方・定番パターン
- **`when` / `each` は必ずクロージャ**：`when=move || cond` のように渡す。直接値を書くと初回固定になり追従しない（signals.md の view ルールと同じ）。
- **`key` は安定 & 一意**：DB の id、UUID など「要素の同一性」を表すものを使う。**インデックス（並び順）を key にしない**（並べ替え・削除でズレる）。
- **単純な真偽分岐は `Show`**、複数分岐は `Show` のネストか、view 内クロージャで `match` した結果を返す形にする。
- **リストの更新は「新しい Vec を set」**：`set_items.update(|v| v.push(...))` のように更新すると、`For` が key を見て差分だけ反映する。
- **空リストの表示**：要素ゼロのときの文言は `Show when=move || !items.get().is_empty()` などで別に出すと素直。

## ハマりどころ / アンチパターン
- **`For` に `key` を付け忘れる / インデックスを key にする**：再描画で要素が作り直され、入力中の状態やフォーカスが飛ぶ。**安定 id を key に**。
- **`each` / `when` を値で渡す**：`when=cond`（×）→ `when=move || cond`（○）。リアクティブに評価されない。
- **巨大リストをそのまま描画**：件数が多いと DOM が重い。フィルタ・ページング（→ Memo で絞る）や仮想化を検討。
- **`Show` の `fallback` を毎回重く作る**：`fallback` も評価されるので、重い view を置きすぎない。
- **要素の `clone()` 漏れ**：`For` の子で文字列等を使うとき所有権で詰まりやすい。`item.label.clone()` のように複製するか、参照設計を見直す。
- **条件描画で状態を破棄したくない場合**：`Show` は偽のとき子を外す。隠すだけにしたいなら CSS（`display`）で制御する手もある。

## 関連
[signals.md](./signals.md) / [effects_memos.md](./effects_memos.md) / [view_macro.md](./view_macro.md)
