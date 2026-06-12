# Leptos 0.x 実務リファレンス（索引）

> **対象 = Leptos 0.7+（Rust / WASM、TS不要）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> ⚠️ Leptos は 0.x 系で **API がよく変わる**（`create_signal`→`signal()`、`create_effect`→`Effect::new` 等）。本書は概念理解が主目的。実装時は最新の公式ドキュメントで確認すること。

## 核となる考え方
```
 signal（値の箱）が変わる → それを使っている view の“その部分だけ”が更新される
 （React のように再レンダリング＆diff はしない＝細粒度リアクティビティ）
```

## 項目（各ファイルへ）

### はじめに / 基礎
- [getting_started.md](./getting_started.md) … 始め方（cargo / trunk / cargo-leptos）
- [components.md](./components.md) … コンポーネント（`#[component]`）とは
- [view_macro.md](./view_macro.md) … `view!` マクロ（テンプレート・イベント）とは

### リアクティビティ
- [signals.md](./signals.md) … signals（リアクティビティの核）とは
- [effects_memos.md](./effects_memos.md) … `Effect` / `Memo`（派生・副作用）とは
- [control_flow.md](./control_flow.md) … 制御フロー（`Show` / `For`）とは

### データ・状態・フルスタック
- [async_resources.md](./async_resources.md) … 非同期データ（`Resource` / `Suspense`）とは
- [server_functions.md](./server_functions.md) … サーバ関数（`#[server]`）とは ← フルスタックの目玉
- [context_state.md](./context_state.md) … グローバル状態（`provide_context` / `use_context`）とは

### 描画・ルーティング・運用
- [request_flow.md](./request_flow.md) … データの流れ（CSR：Signal→viewの一方向／SSR：server function→HTML）・各部分が何を返すか
- [ssr_csr.md](./ssr_csr.md) … 描画モード（CSR / SSR / ハイドレーション）とは
- [routing.md](./routing.md) … ルーティング（leptos_router）とは

### テスト
- [wasm_bindgen_test.md](./wasm_bindgen_test.md) … wasm-bindgen-test（ブラウザ/WASM上のDOMテスト）とは

- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Leptos）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
