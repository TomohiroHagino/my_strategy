# 描画モード（CSR / SSR / ハイドレーション）（Leptos）

## ひとことで言うと
同じ Leptos アプリを「ブラウザだけで動かす（**CSR**）」のか、「サーバで HTML を生成してから動かす（**SSR ＋ ハイドレーション**）」のか、という**実行モードの切り替え**。どのモードで動くかは **feature フラグ**（`csr` / `ssr` / `hydrate`）で決まる。

## 役割・なぜ必要か
- **CSR (Client-Side Rendering)**：サーバはほぼ空の HTML を返し、WASM が起動してから画面を組み立てる。構成が単純で、`trunk` だけで完結する。SPA・社内ツール・静的ホスティング向き。
- **SSR (Server-Side Rendering) ＋ ハイドレーション**：サーバ側でまず HTML を文字列として描画して返す（初期表示が速い・SEO に強い）。その HTML をブラウザに送ったあと、クライアントの WASM が同じ view ツリーに「**ハイドレーション**（hydrate＝対話化）」して、イベントや signal を後付けで結びつける。`cargo-leptos` が前提。
- つまり SSR では**サーバ用とクライアント用、2 つのバイナリ**（ターゲット）を 1 つのコードベースから作る。その出し分けが feature フラグ。

## 基本の書き方（コード）
```toml
# Cargo.toml（SSR 構成の例。trunk 単体の CSR なら csr のみ）
[features]
csr = ["leptos/csr"]
ssr = ["leptos/ssr", "dep:axum", "dep:tokio", "leptos_axum"]
hydrate = ["leptos/hydrate"]
```

```rust
// サーバ専用コードは cfg で gate（クライアント WASM には含めない）
#[cfg(feature = "ssr")]
pub async fn db_pool() -> sqlx::PgPool {
    // DB 接続などサーバ側だけの処理
    todo!()
}

// SSR の起動側（Axum 統合の骨組み。Actix なら leptos_actix を使う）
#[cfg(feature = "ssr")]
#[tokio::main]
async fn main() {
    use axum::Router;
    use leptos_axum::{generate_route_list, LeptosRoutes};
    let conf = leptos::config::get_configuration(None).unwrap();
    let routes = generate_route_list(App);
    let app = Router::new()
        .leptos_routes(&conf.leptos_options, routes, App);
    // axum::serve(...).await で起動
}

// hydrate ターゲットのエントリ（クライアント側で対話化）
#[cfg(feature = "hydrate")]
#[wasm_bindgen::prelude::wasm_bindgen]
pub fn hydrate() {
    leptos::mount::hydrate_body(App);
}
```

## 実務での使い方・定番パターン
- **まず CSR で試す**：`trunk serve` だけで動くので学習・PoC が速い。サーバ機能（`#[server]`・DB）が要らないならこれで十分。→ [getting_started.md](./getting_started.md)
- **フルスタックにするなら SSR**：`cargo leptos watch` で開発。サーバ関数を使う／SEO が要る／初期表示を速くしたい、なら SSR を選ぶ。→ [server_functions.md](./server_functions.md)
- **サーバ専用処理は必ず `#[cfg(feature = "ssr")]` で囲う**：DB クライアント・ファイル I/O・秘密鍵の参照などをクライアント WASM に混ぜないため。サーバ関数の本体も同様に gate されるのが基本。
- **Axum / Actix 統合**：`leptos_axum` / `leptos_actix` がルーティングとレスポンス生成を橋渡しする。テンプレート（`cargo leptos new`）から始めると配線済みで楽。

## ハマりどころ / アンチパターン
- **feature フラグの設定ミス**：`ssr` と `hydrate` を同時に有効化したり、`default` に紛れ込ませると、サーバ向けとクライアント向けのビルドが混線してリンクエラーや実行時パニックになる。フラグは「サーバビルド＝`ssr`、クライアントビルド＝`hydrate`」の出し分けが要点。
- **`web-sys` 等ブラウザ API を SSR で呼ぶ**：`window` / `document` / `localStorage` などはサーバ側に存在しない。トップレベルで叩くと SSR 描画時にパニックする。**`Effect` の中**（クライアントでしか走らない箇所）や `create_node_ref` 経由で触る。→ [effects_memos.md](./effects_memos.md)
- **ハイドレーションミスマッチ**：サーバが描いた HTML と、クライアントが期待する view が食い違うと、ハイドレーションが壊れ画面が無反応・重複描画になる。原因は「サーバとクライアントで値が変わる処理（乱数・現在時刻・`web-sys` 依存の分岐）」を描画に使うこと。初期 view は両者で同一になるよう保つ。
- **クライアント専用クレートをサーバビルドに入れる**：`wasm-bindgen` 前提のクレートを `ssr` 側にも含めるとビルドが通らない。依存も feature で分ける。

## 関連: [server_functions.md](./server_functions.md) / [getting_started.md](./getting_started.md)
