# コンポーネント（Leptos）

> ⚠️ Leptos 0.x は API が変わりやすい。本書は **0.7+** 前提。細部は最新 docs で確認。

## ひとことで言うと
**`#[component]` を付けた関数**で、UI の部品1つを表す。戻り値は `impl IntoView`（描画できる何か）。React の関数コンポーネントに近いが、**呼ばれるのは一度だけ**（再レンダリングしない）。

## 役割・なぜ必要か
- UI を「ボタン」「カード」「フォーム」のような **再利用できる単位** に分割するため。
- Leptos は細粒度リアクティビティなので、コンポーネントは **「最初に1回だけ実行され、view を組み立てる」設計図**。以後の更新は signal が担当し、コンポーネント関数自体は再実行されない（ここが React と決定的に違う）。→ [signals.md](./signals.md)
- 関数の引数 = **props（親から渡す値）**。型で受け取るので、渡し間違いはコンパイルエラーで弾ける。

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

// PascalCase の関数名にする（view! 内で <Button/> と書けるようにするため）
#[component]
fn Button() -> impl IntoView {
    view! { <button>"クリック"</button> }
}

// props は関数引数で受け取る
#[component]
fn Greeting(
    name: String,                       // 必須の prop
    #[prop(optional)] suffix: String,   // 省略可（無ければ String::default()）
    #[prop(into)] count: i32,           // 渡す側の型を自動変換して受け取る
) -> impl IntoView {
    view! { <p>{name}{suffix}" : "{count}</p> }
}

// 子要素を受け取る（<Card>...</Card> の ... 部分）
#[component]
fn Card(children: Children) -> impl IntoView {
    view! { <div class="card">{children()}</div> }
}

// 使う側
#[component]
fn App() -> impl IntoView {
    view! {
        <Button/>
        <Greeting name="太郎".to_string() count=3/>
        <Card>
            <p>"カードの中身"</p>
        </Card>
    }
}
```

## 実務での使い方・定番パターン
- **小さく分ける**。1コンポーネント=1責務。大きくなったら子コンポーネントへ切り出す。
- **`#[prop(into)]`** を多用する：呼ぶ側が `"abc"`（`&str`）を書いても `String` に自動変換され、記述が軽くなる。文字列・signal を渡す props で定番。
- **`#[prop(optional)]`** で省略可能な props を作り、デフォルト値を許容する。`#[prop(default = ...)]` で既定値も指定できる。
- **`children: Children`** でラッパー系コンポーネント（レイアウト、カード、モーダル）を作る。`children()` と **呼び出して** 展開する点に注意。
- リアクティブな値を渡したいときは **signal を prop として渡す**（値のコピーではなく「箱」を渡す）。→ [signals.md](./signals.md)
- グローバルな状態は props のバケツリレーにせず context で配る。→ [context_state.md](./context_state.md)

## ハマりどころ / アンチパターン
- **`#[component]` の付け忘れ**：ただの関数になり `view!` 内で `<Foo/>` として使えない。
- **関数名が小文字（snake_case）**：`<button/>` と区別がつかず、HTML タグ扱いされて意図通り描画されない。**コンポーネントは PascalCase**。
- **コンポーネントが再実行されると思い込む**：React の癖で「props が変わったら関数がまた走る」と考えるとハマる。Leptos は **1回だけ実行**。値の変化に追従させたい部分は signal とクロージャで書く。→ [view_macro.md](./view_macro.md)
- **`children` をただ `{children}` と書く**：`Children` は呼び出して使うので `{children()}` が正しい。
- props に **巨大な値の move を毎回渡す** より、変化するものは signal で渡す方がリアクティブに扱える。
- props 属性（`into` / `optional` / `default`）の付け忘れで、呼ぶ側のコードが冗長になる・型が合わない。

## 関連
[view_macro.md](./view_macro.md) / [signals.md](./signals.md) / [context_state.md](./context_state.md)
