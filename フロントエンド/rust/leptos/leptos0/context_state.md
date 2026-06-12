# グローバル状態（`provide_context` / `use_context`）（Leptos）

> ⚠️ 0.x 系は API 変動が大きい。本書は **0.7 系**（`provide_context` / `use_context`）前提。実装時は最新の公式ドキュメントで確認すること。

## ひとことで言うと
`provide_context(value)` は **親コンポーネントで値を「配置」**し、`use_context::<T>()` で **子孫コンポーネントがその値を「型で取り出す」**仕組み。間のコンポーネントに props を順送りせずに、ツリーの下まで値を届けられる。

## 役割・なぜ必要か
- 「現在ログイン中のユーザ」「テーマ」「グローバル設定」など、**多くのコンポーネントが必要とする値**を、いちいち props でバケツリレー（prop drilling）するのは煩雑で壊れやすい。
- `provide_context` で**祖先に1回置けば、子孫はどこからでも取得**できる。React の Context に近い発想。
- 状態を **変更可能にしたい**場合は、ただの値ではなく `RwSignal<T>` を context で配る。すると「context 越しに read も write もできるグローバル状態」になり、変更が細粒度リアクティビティで関係箇所だけに伝わる。→ [signals.md](./signals.md)

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

// context で配る型（型そのものが「鍵」になる。同じ型を複数配るなら newtype で区別）
#[derive(Clone, Copy)]
struct ThemeState(RwSignal<bool>); // true = dark

#[component]
fn App() -> impl IntoView {
    // 親で「提供」する
    let dark = RwSignal::new(false);
    provide_context(ThemeState(dark));

    view! { <Toolbar/> <Content/> }
}

#[component]
fn Toolbar() -> impl IntoView {
    // 子で「取得」する → Option<T>（無ければ None）
    let theme = use_context::<ThemeState>()
        .expect("ThemeState が provide されていません");

    view! {
        <button on:click=move |_| theme.0.update(|d| *d = !*d)>
            "テーマ切替"
        </button>
    }
}

#[component]
fn Content() -> impl IntoView {
    let theme = expect_context::<ThemeState>(); // 必須前提なら expect_context が簡潔
    view! {
        <p>{move || if theme.0.get() { "ダーク" } else { "ライト" }}</p>
    }
}
```

`use_context::<T>()` は **`Option<T>`** を返す（提供されていなければ `None`）。「必ずあるはず」なら `expect_context::<T>()` が簡潔（無ければパニック）。

## 実務での使い方・定番パターン
- **グローバル状態＝`RwSignal` を context で配る。** 値そのものではなく signal を配ることで、子孫から更新でき、更新が必要な箇所にだけ伝播する。
- **型で引く設計にする。** context は「型」が鍵。同じ `i32` を2つ配ると後勝ちで混乱するので、`struct UserId(RwSignal<u32>)` のように **newtype で意味づけ**して区別する。
- **提供スコープを意識する。** `provide_context` した子孫だけが取得できる。ルート付近（`App`）で配れば全体で使える。
- **サーバ側 context** にも同じ仕組みを使う。`#[server]` 内で `use_context::<DbPool>()` のように、リクエスト単位の依存（DB プール等）を受け取る。→ [server_functions.md](./server_functions.md)
- **読みやすさのために薄いラッパ関数**（`fn use_theme() -> ThemeState { expect_context() }`）を用意すると、取得箇所が統一できる。

## ハマりどころ / アンチパターン
- **`use_context` が `Option` なのを忘れる。** 提供し忘れ／スコープ外だと黙って `None`。`unwrap()` でパニックするか、`expect_context` で「未提供」と明示エラーにする。
- **同じ型を複数 provide** して取り違える。必ず newtype で型を分ける。型一致だけが手がかり。
- **何でも context に置く**＝事実上のグローバル変数化でテスト・追跡が困難に。本当に広く共有する値だけに絞り、局所的な状態は props や近い親の signal で済ませる。
- **provide より下でしか取れない**。兄弟や上位では取得できないので、提供位置（どの祖先で `provide_context` するか）を設計段階で決める。
- **`Copy` でない値**を context に入れて取り回しに苦労する。`RwSignal` は `Copy` なので扱いやすい。重い値は signal 経由で持つとよい。

## 関連
[signals.md](./signals.md) / [components.md](./components.md) / [server_functions.md](./server_functions.md)
