# signals（リアクティビティの核）（Leptos 0.7+）

## ひとことで言うと
**値の「箱」**。中身が変わると、その値を**使っている view のその部分だけ**が自動で更新される（React のような再レンダリング＆diff はしない）。Leptos のリアクティビティの最小単位。

## 役割・なぜ必要か
- UI を「状態（signal）の関数」として書くための土台。状態を変えれば描画が追従する。
- **細粒度リアクティビティ**＝「どの signal が、view のどこで使われたか」を実行時に記録し、変更時はその**依存箇所だけ**を更新する。仮想 DOM の差分計算を持たないので、無駄な再評価が起きにくい。
- 値の「読み（リアクティブに購読）」と「書き」を型で分けることで、どこが状態を変えうるかが追いやすい。

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

#[component]
fn Counter() -> impl IntoView {
    // タプルで読み手 / 書き手を分けて受け取る
    let (count, set_count) = signal(0);

    view! {
        // view 内では「クロージャ」で渡すとリアクティブに購読される
        <p>"現在: " {move || count.get()}</p>

        // set: 値を置き換える
        <button on:click=move |_| set_count.set(0)>"リセット"</button>

        // update: 今の値を受け取って更新（インクリメント等に向く）
        <button on:click=move |_| set_count.update(|n| *n += 1)>"+1"</button>
    }
}
```

`count.get()` は**リアクティブに購読**する（呼ばれた場所が依存先として登録される）。
購読したくない（一度だけ読む）ときは `count.get_untracked()`。

```rust
// 読み手 / 書き手をまとめた 1 個の signal が欲しいとき
let count = RwSignal::new(0);
count.get();              // 読み（購読する）
count.set(1);             // 書き（置き換え）
count.update(|n| *n += 1);// 書き（更新）
```

`(read, write)` 版と `RwSignal` 版は用途で選ぶ。
- **`signal()`（タプル）**：読み専用・書き専用を別々に配れるので「誰が書けるか」が明確。
- **`RwSignal::new()`**：1 個で読み書き両方。コンテキストで共有する状態などに便利。

## 実務での使い方・定番パターン
- **view 内は必ずクロージャで**：`{move || count.get()}`。`{count.get()}` と直接書くと**初回値が固定**され、更新に追従しない。
- **インクリメント系は `update`**：`set_count.set(count.get() + 1)` でも動くが、`update(|n| *n += 1)` の方が「読んでから書く」競合がなく素直。
- **`get_untracked` は限定用途**：Effect 内などで「今の値は欲しいが、この読みでは購読したくない」場合に使う。多用すると更新が来なくなるバグ源。
- **派生値は signal にしない**：`count * 2` のような値はクロージャ `move || count.get() * 2` か `Memo`（→ [effects_memos.md](./effects_memos.md)）で表す。状態を二重に持つと同期ズレの元。
- **`Copy` 前提で扱える**：signal ハンドルは `Copy`。`move` クロージャに何度でも持ち込めるので、所有権で悩む場面は少ない。

## ハマりどころ / アンチパターン
- **view で `get()` を直書き**：`{count.get()}` は静的。**`{move || count.get()}`** にする。
- **旧 API を使う**：`create_signal` は 0.7 系では**非推奨/廃止**方向。**`signal()`**（または `RwSignal::new`）を使う。
- **`set` と `update` の混同**：`set` は置き換え、`update` は今の値を編集。前の値を使うなら `update`。
- **`get_untracked` の多用**：購読しないので「変えたのに画面が変わらない」。理由がある時だけ使う。
- **状態の重複**：同じ事実を 2 つの signal に持つと整合が崩れる。**単一の出どころ**にし、派生はクロージャ/Memo で。
- **`set` の中で同じ signal を `get`**：無限ループや読み書き競合の温床。`update` のクロージャ引数で受けるのが安全。

## 関連
[effects_memos.md](./effects_memos.md) / [control_flow.md](./control_flow.md) / [view_macro.md](./view_macro.md)
