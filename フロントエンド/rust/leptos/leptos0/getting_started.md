# はじめに / 始め方（Leptos）

> ⚠️ Leptos は 0.x 系で API がよく変わる。本書は **0.7+** を前提に書くが、実装時は最新の公式 docs で確認すること。

## ひとことで言うと
Rust で書いた UI を **WASM にコンパイルしてブラウザで動かす** ためのセットアップ手順。Rust 本体 + `wasm32` ターゲット + ビルドツール（`trunk` か `cargo-leptos`）の3点を揃える。

## 役割・なぜ必要か
- ブラウザは Rust を直接実行できない。**Rust → WebAssembly（wasm）** に変換し、それを JS から起動する必要がある。そのためのターゲットとツールチェーンを最初に用意する。
- ビルドツールは2系統あり、用途で選ぶ：
  - **`trunk`** … CSR（ブラウザだけで動く SPA）向け。`index.html` を起点にビルド。手軽。
  - **`cargo-leptos`** … SSR / フルスタック（サーバ側レンダリング + `#[server]` 関数）向け。サーバとクライアントを一括ビルド。
- 「まず動かして手触りを掴む」なら CSR + trunk から始めるのが最短。

## 基本の書き方（コード）
```bash
# 1) Rust 導入（未導入なら）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 2) wasm ターゲットを追加（これを忘れるとビルドできない）
rustup target add wasm32-unknown-unknown

# 3) ビルドツール。CSR なら trunk
cargo install trunk
# SSR / フルスタックなら cargo-leptos
cargo install cargo-leptos
```

```toml
# Cargo.toml （CSR の最小構成）
[package]
name = "leptos0"
version = "0.1.0"
edition = "2021"

[dependencies]
leptos = { version = "0.7", features = ["csr"] }
```

```rust
// src/main.rs
use leptos::prelude::*;

#[component]
fn App() -> impl IntoView {
    view! { <h1>"Hello, Leptos!"</h1> }
}

fn main() {
    // CSR: <body> 直下にマウントして描画開始
    mount_to_body(App);
}
```

```html
<!-- index.html （trunk の起点。プロジェクト直下に置く） -->
<!DOCTYPE html>
<html>
  <head></head>
  <body></body>
</html>
```

```bash
# 開発サーバ起動（自動リロード付き）
trunk serve --open
```

## 実務での使い方・定番パターン
- **CSR で素振り → 必要になったら SSR へ**。最初からフルスタックを組むと設定が重い。学習段階は trunk が楽。
- **feature フラグで描画モードを切り替える**：`csr` / `ssr` / `hydrate` のどれを有効にするかで挙動が変わる。フルスタックでは「サーバビルドは `ssr`、クライアントビルドは `hydrate`」のように使い分け、`cargo-leptos` がこれを面倒みてくれる。→ [ssr_csr.md](./ssr_csr.md)
- `leptos = { version = "0.7", features = ["csr"] }` のように **必ず1つの描画 feature を有効にする**。無いとマウント系 API が使えない。
- スタイルは trunk なら `index.html` に CSS を読み込む、もしくは Tailwind 等を組み合わせる構成が定番。

## ハマりどころ / アンチパターン
- **`wasm32-unknown-unknown` の追加忘れ**が最頻出。`error: target ... not found` 系が出たらまずこれを疑う。
- **trunk と cargo-leptos を取り違える**：CSR なのに cargo-leptos を使おうとして設定が合わない、逆に SSR なのに trunk で `#[server]` が動かない、など。**用途で選ぶ**。
- **feature フラグの付け忘れ / 二重付け**：`csr` と `ssr` を同時に有効化すると衝突する。1ビルド1モードが基本。
- `use leptos::prelude::*;` を忘れて `view!` や `mount_to_body` が見つからない、というつまずきが多い（0.7 は prelude 経由が基本）。
- バージョン混在に注意：`leptos` 本体と `leptos_router` 等の **マイナーバージョンを揃える**。0.6 系と 0.7 系で API が違う。

## フォルダ構成（始動直後）
```
myapp/
├── src/
│   ├── main.rs             # 起動（CSRはmount_to_body、SSRはサーバ起動）
│   ├── lib.rs              # ライブラリ起点（コンポーネント公開）
│   └── app.rs              # App コンポーネント（view! マクロ）
├── Cargo.toml              # 依存・leptos設定（[package.metadata.leptos]）
├── Cargo.lock              # 依存の固定（自動生成）
├── style/
│   └── main.scss           # スタイル（cargo-leptos が処理）
├── public/
│   └── favicon.ico         # 静的ファイル
├── end2end/                # E2Eテスト（cargo-leptos 雛形に含まれる）
├── index.html              # Trunk の起点（# Trunk構成のとき自分で作る）
├── Trunk.toml              # Trunk の設定（# Trunk構成のとき自分で作る）
└── target/                 # ビルド成果物（cargo が生成）
```
- SSR/フルスタックは cargo-leptos（leptos設定は Cargo.toml 内）。CSRのみなら Trunk（index.html 起点）。コンポーネントは `view!` マクロで書く。

## 関連
[components.md](./components.md) / [ssr_csr.md](./ssr_csr.md)
