# 実務でハマる罠まとめ（Pitfalls）（Leptos 0.7+）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Leptos は 0.x 系で **API がよく変わる**うえ、CSR / SSR / hydrate でビルドが分岐する。症状から該当箇所へ素早く飛ぶための索引が要る。
- ⚠️ 本書全体の大前提：**0.x で API が変わる**。書き方が合わない時はまず最新の公式ドキュメント・例で確認すること。

## signal / リアクティビティ
- **signal の読み書き**：読みは `.get()`、書きは `.set()` / `.update()`（部分更新）。view 内の `{}` では `move || count.get()` のクロージャで包むのが基本。→ [signals.md](./signals.md)
- **旧 `create_*` API は変わった**：`create_signal`→`signal()`、`create_effect`→`Effect::new`、`create_resource`→`Resource::new`、`create_memo`→`Memo::new`。古い記事のコードをそのまま写すと通らない。→ [signals.md](./signals.md) / [effects_memos.md](./effects_memos.md)
- **動的値はクロージャで包む**：view に値を直接埋めると「最初の値で固定」され更新されない。`{move || ...}` で包むと signal 変化に追従する。→ [view_macro.md](./view_macro.md)
- **`<For>` には `key` 必須**：リスト描画は要素を一意に識別する `key` を指定しないと、再描画で取り違え・状態のズレが起きる。→ [control_flow.md](./control_flow.md)

## サーバ / フルスタック
- **サーバ関数の戻り値制約**：`#[server]` 関数の入出力は `Serialize`/`Deserialize` 可能な型で、戻りは `Result<T, ServerFnError>`。境界を越える型がシリアライズ不可だとビルドや実行で失敗する。→ [server_functions.md](./server_functions.md)
- **feature フラグ（csr / ssr / hydrate）の設定**：サーバビルド＝`ssr`、クライアントビルド＝`hydrate` の出し分けがずれると、リンクエラーや実行時パニックになる。→ [ssr_csr.md](./ssr_csr.md)
- **`web-sys` は SSR で不可 / ハイドレーションミスマッチ**：`window`/`document` 等はサーバに無く、トップレベルで呼ぶと SSR でパニック。サーバとクライアントで初期 view が食い違うとハイドレーションが壊れる。`Effect` 内で触り、初期描画は両者一致させる。→ [ssr_csr.md](./ssr_csr.md)

## 状態 / ルーティング / 環境
- **`use_context` は `Option`**：`provide_context` し忘れ・スコープ外だと `None`。`expect()` で握らず欠落を分岐する。→ [context_state.md](./context_state.md)
- **`wasm32` ターゲット追加忘れ**：`rustup target add wasm32-unknown-unknown` をしないとクライアントビルドが通らない。`trunk`/`cargo-leptos` も導入する。→ [getting_started.md](./getting_started.md)
- **0.7 でルーティング API 変更**：パスは `path!("/users/:id")` マクロで書く。リンクは `<a>` でなく `<A>`。`<Routes>` に `fallback` を付ける。→ [routing.md](./routing.md)

## 関連
[signals.md](./signals.md) / [effects_memos.md](./effects_memos.md) / [view_macro.md](./view_macro.md) / [control_flow.md](./control_flow.md) / [server_functions.md](./server_functions.md) / [ssr_csr.md](./ssr_csr.md) / [context_state.md](./context_state.md) / [routing.md](./routing.md) / [getting_started.md](./getting_started.md)
