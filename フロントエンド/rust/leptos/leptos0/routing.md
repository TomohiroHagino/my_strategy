# ルーティング（leptos_router）（Leptos）

## ひとことで言うと
URL のパスと、表示するコンポーネント（view）を対応づける仕組み。`leptos_router` クレートが提供する `<Router>` / `<Routes>` / `<Route>` で「**このパスならこの view**」を宣言的に定義する。SPA でもページ遷移を URL に乗せられる。

## 役割・なぜ必要か
- 1 ページのアプリでも「一覧」「詳細」「設定」のように**画面を URL で切り替えたい**。ルーティングはその対応表。
- ブラウザの戻る／進む・URL 直打ち・ブックマーク・共有 URL を、フルリロードなしで成立させる（CSR でも SSR でも同じ書き味で動く）。
- パスのパラメータ（`/users/:id` の `id`）を view から取り出せるので、URL を「状態」として扱える。

## 基本の書き方（コード）
```rust
use leptos::prelude::*;
use leptos_router::components::{Router, Routes, Route, A};
use leptos_router::{path, hooks::use_params, params::Params};

#[derive(Params, PartialEq, Clone)]
struct UserParams {
    id: Option<String>,
}

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <nav>
                // 遷移リンクは <a> ではなく <A>
                <A href="/">"Home"</A>
                <A href="/users/1">"User 1"</A>
            </nav>
            <main>
                <Routes fallback=|| "Not found">
                    <Route path=path!("/") view=Home/>
                    // :id は動的セグメント（パラメータ）
                    <Route path=path!("/users/:id") view=User/>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
fn User() -> impl IntoView {
    let params = use_params::<UserParams>();
    let id = move || params.get().ok().and_then(|p| p.id).unwrap_or_default();
    view! { <p>"User id = " {id}</p> }
}
```

## 実務での使い方・定番パターン
- **`use_params` でパラメータ取得**：`/users/:id` の `id` を `Params` 派生の構造体に束ねて読む。結果は `Result`／フィールドは `Option` なので「無い場合」を必ず処理する。
- **`use_navigate` でプログラム遷移**：フォーム送信後などコードから移動したいとき。`let nav = use_navigate(); nav("/done", Default::default());` のように呼ぶ。
- **`<A>` でリンク**：`<A href="/path">` はクライアント遷移（フルリロードなし）。外部リンクや特別な挙動が要るときだけ生の `<a>` を使う。
- **ネストルート**：親 `<Route>` の中に子 `<Routes>`／`<Outlet/>` を置き、共通レイアウト（サイドバー等）を保ったまま中身だけ差し替える。
- **クエリ**：`use_query` で `?key=value` を、`use_location` で現在の location 情報を読む。フィルタやページ番号を URL に持たせる用途に向く。

## ハマりどころ / アンチパターン
- **0.7 でルーティング API が変わった**：パスは文字列直書きでなく **`path!("/users/:id")` マクロ**で書く形に変わった（型安全・セグメント解釈のため）。古い記事の `<Route path="/users/:id" ...>` の書き方をそのまま写すと型が合わない。最新ドキュメントで確認する。
- **リンクに `<a>` を使ってしまう**：素の `<a href>` はフルページリロードになり、SPA の状態が飛ぶ。アプリ内遷移は必ず **`<A>`**。
- **`<Routes>` に `fallback` 未指定**：マッチしないパスで何も出ない／パニックの原因になる。`fallback=|| ...` で 404 相当を必ず用意する。
- **パラメータを直接信用する**：`use_params` の値は `Option`／`Result`。`unwrap()` で握ると URL 改変や直打ちで落ちる。空・不正値を分岐して扱う。→ [components.md](./components.md)
- **ネストルートで `<Outlet/>` を置き忘れる**：親側に出力口がないと子 view が描画されない。

## 関連: [components.md](./components.md)
