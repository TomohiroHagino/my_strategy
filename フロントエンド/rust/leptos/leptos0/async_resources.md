# 非同期データ（`Resource` / `Suspense`）（Leptos）

> ⚠️ 0.x 系は API 変動が大きい。本書は **0.7 系**（`Resource::new`）前提。実装時は最新の公式ドキュメントで確認すること。

## ひとことで言うと
`Resource` は **「source（依存する signal）が変わるたびに、async な fetcher を再実行して結果を保持する」非同期データの箱**。fetch 中はまだ値が無いので、`<Suspense>` で「読込中の表示」を出しながら待つ。

## 役割・なぜ必要か
- WASM 上の Leptos では DB アクセスや API 呼び出しは **非同期（`Future`）**。signal は同期的な値の箱なので、そのままでは「await した結果」を view に流し込めない。
- `Resource` は **「いつ再 fetch するか（source）」と「どう fetch するか（fetcher）」を分離**して持つ。source が変われば自動で fetcher を呼び直し、結果を最新化してくれる。
- これにより「ユーザIDが変わったらそのユーザを取り直す」のような **依存に応じた再取得** を、手続き的に書かずに宣言できる。
- SSR では、サーバ側で fetch を済ませて HTML に焼き込み、クライアントでハイドレートする橋渡しも担う。→ [server_functions.md](./server_functions.md)

## 基本の書き方（コード）
```rust
use leptos::prelude::*;

#[component]
fn UserCard() -> impl IntoView {
    // source（依存）になる signal
    let (user_id, set_user_id) = signal(1u32);

    // Resource::new(source, fetcher)
    //   第1引数 = 依存を読む同期クロージャ（ここが変わると再 fetch）
    //   第2引数 = source の値を受け取る async クロージャ（fetcher）
    let data = Resource::new(
        move || user_id.get(),                       // source
        |id| async move { fetch_user(id).await },    // fetcher
    );

    view! {
        <button on:click=move |_| set_user_id.update(|n| *n += 1)>
            "次のユーザ"
        </button>

        // fetch 中は fallback、揃ったら子を描画
        <Suspense fallback=move || view! { <p>"読込中…"</p> }>
            {move || data.get().map(|user| view! {
                <p>"名前: " {user.name}</p>
            })}
        </Suspense>
    }
}

// 例：実際は server function を呼ぶことが多い
async fn fetch_user(id: u32) -> User {
    // ... await ...
    User { name: format!("user-{id}") }
}

#[derive(Clone)]
struct User { name: String }
```

`Resource::get()` は **`Option<T>`**（まだ来ていなければ `None`）。`Suspense` 内で `.get()` を読むと、Leptos が「この Resource を待つ」と認識する。

## 実務での使い方・定番パターン
- **source は「再取得のトリガになる signal だけ」を読む。** source 内で関係ない signal を読むと無駄な再 fetch が起きる。
- **`<Suspense fallback=… >` で必ず包む。** 中の `move || data.get()` が `None` の間 fallback が出る。複数 Resource を同じ Suspense に入れれば「全部揃うまで待つ」になる。
- **エラーも扱うなら fetcher の戻り値を `Result` に**し、`<Suspense>` の内側にさらに `<ErrorBoundary>` を組む。
- **再 fetch せず一度きりの取得**でいいなら、source を `|| ()` にする（依存なし）。
- **`<Await>`**：source が無く「1回 await して描画」したいだけのときの簡易版。
```rust
view! {
    <Await future=fetch_user(1) let:user>
        <p>{user.name.clone()}</p>
    </Await>
}
```
`<Await>` は再取得しない単発用、依存に応じた再取得が要るなら `Resource` を使う、と覚えると分かりやすい。

## ハマりどころ / アンチパターン
- **source と fetcher を混同する。** source は同期で「依存を読むだけ」、fetcher は async で「実際に取りに行く」。fetcher 内で signal を `.get()` してもそれは依存として追跡されない（追跡されるのは source 側）。
- **`<Suspense>` で包み忘れる。** `data.get()` が `None` のまま `unwrap()` してパニック、あるいは何も出ない。非同期の値は Suspense（または Await）越しに読むのが基本。
- **source で重い計算や無関係な signal を読む** → 想定外の再 fetch ループ。source は最小限に。
- **SSR との絡み**：server function を fetcher にすると、SSR 時はサーバで実行され HTML に値が入る。CSR のみの構成と挙動が変わるので、描画モードを意識する。→ [ssr_csr.md](./ssr_csr.md)
- `Resource::get()` の `Option` を「エラー」と取り違えない。`None` は「まだ来ていない」であって失敗ではない。失敗は fetcher の `Result` で表す。

## 関連
[server_functions.md](./server_functions.md) / [signals.md](./signals.md) / [ssr_csr.md](./ssr_csr.md)
