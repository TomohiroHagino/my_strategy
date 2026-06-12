# wasm-bindgen-test（Leptos）

## ひとことで言うと
**WASM をブラウザ（または headless ブラウザ）上で実行してテストするための仕組み**。`#[wasm_bindgen_test]` を付けた関数を `wasm-pack test` で走らせ、本物の DOM・`window`・JS API がある環境でテストできる。Leptos のコンポーネントは描画して DOM を作るため、DOM が無いと検証できない部分をこれで担当する。

## 役割・なぜ必要か
- Rust 標準の `#[test]` / `cargo test` は **ネイティブ実行**で DOM が無い。純粋なロジック（signal の計算・データ整形）には十分だが、`view!` の描画結果や DOM 操作は検証できない。
- Leptos の本質は「signal が変わると view のその部分だけ更新される」。**実際に DOM が更新されたか**を見るには、DOM のあるブラウザ/WASM 環境が要る。
- `#[wasm_bindgen_test]` は WASM へコンパイルして **本物のブラウザ API 上で実行**するので、`document` から要素を取り出して内容・属性を検証できる。
- `wasm-bindgen-test-runner` が headless Chrome / Firefox を起動し、CI でも回せる。

## 基本の書き方（コード）
ロジックは標準テストで十分（DOM 不要）。
```rust
// src/math.rs
pub fn next_count(n: i32) -> i32 { n + 1 }

#[cfg(test)]
mod tests {
    use super::*;
    #[test] // 普通の cargo test。DOMは要らない
    fn increments() {
        assert_eq!(next_count(0), 1);
    }
}
```
DOM を伴う検証は `#[wasm_bindgen_test]`（`tests/` 配下）。
```rust
// tests/web.rs
use wasm_bindgen_test::*;
wasm_bindgen_test_configure!(run_in_browser); // ブラウザで実行する宣言

#[wasm_bindgen_test]
fn renders_into_dom() {
    // 本物の document が使える
    let document = web_sys::window().unwrap().document().unwrap();
    let body = document.body().unwrap();

    // ここで Leptos の view をマウントして body に描画する想定
    // mount_to(body.clone().into(), || view! { <p class="count">"1"</p> });

    let el = document.query_selector(".count").unwrap().unwrap();
    assert_eq!(el.text_content().unwrap(), "1");
}
```
実行：
```bash
wasm-pack test --headless --chrome   # headless Chrome で WASM テストを走らせる
```

## 実務での使い方・定番パターン
- **テストを2層に分ける**：
  - **ロジック（純関数・signal の計算）** → 速い `#[test]` / `cargo test`。大半をここに置く。
  - **描画・DOM 更新・イベント** → 少数の `#[wasm_bindgen_test]`。重要な箇所だけ。
- **`run_in_browser`**：DOM（`web_sys`）を使うなら必須。付けないと Node 実行になり `document` が無い。
- **非同期テスト**：`Resource` / `Suspense` の解決を見るときは `async` テストにできる。
```rust
#[wasm_bindgen_test]
async fn loads_async_data() {
    // await で非同期取得の解決を待ってから DOM を検証
}
```
- **重い E2E は専用ツールへ**：複数ページ遷移・実 API を通す本格 E2E は、ビルドしたアプリに対し Playwright 等で外から叩く方が安定。`wasm-bindgen-test` は **コンポーネント/DOM 単位** に向く。

## ハマりどころ
- **`run_in_browser` 漏れ**：DOM が無い実行環境になり `window`/`document` が `None` で落ちる。ブラウザ実行を明示する。
- **ブラウザ未インストール**：`--headless --chrome` などに対応するブラウザ/ドライバが CI に無いと起動失敗。環境に用意する。
- **ロジックまで WASM テストにする**：遅くて取り回しが悪い。DOM 不要なものは `#[test]` に置く。
- **API の流動性**：Leptos 0.x はマウント API（`mount_to` 等）が版で変わる。実装時は使用バージョンの公式ドキュメントで確認する。
- **DOM のクリーンアップ漏れ**：前テストが残した要素を拾って順序依存で落ちる。テストごとに対象を限定/掃除する。

## 関連
[components.md](./components.md) / [signals.md](./signals.md) / [async_resources.md](./async_resources.md) / [ssr_csr.md](./ssr_csr.md)
