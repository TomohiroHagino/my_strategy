# Rust（フロントエンド / WASM）

## 一言で
**Rust で書いたコードを WebAssembly(WASM) にコンパイルして、ブラウザで動かす**領域。型安全・高速・Rustバックエンドと資産共有できるのが魅力。ただし**エコシステムは新しく・ニッチ**で、Web全体ではまだ JS/TS が主流。

## WASM とは（前提）
- ブラウザで動く**バイナリ命令フォーマット**。JS以外の言語（Rust/C++/Go等）を高速に実行できる。
- Rustフロントは「Rust → WASM → ブラウザ」。DOM操作は `web-sys`/`wasm-bindgen` 経由。

## 主要フレームワーク
| | 特徴 | たとえると |
|---|---|---|
| **Leptos** | 細粒度リアクティブ（signals）・SSR/フルスタック強い・高速 | SolidJS の Rust 版 |
| **Yew** | コンポーネント＋仮想DOM・最も成熟 | React の Rust 版 |
| **Dioxus** | クロスプラットフォーム（Web/Desktop/Mobile） | React Native 的 |

## 強み / 弱み
- 強み：型安全・高性能・Rustでフロントもバックも統一・WASMで重い処理が速い。
- 弱み：**学習コスト高・エコシステム/求人が少ない・0.x で変化が速い**・ビルドが重い・JSとの相互運用が要る場面も。基本はまだ JS/TS が現実解。

## このフォルダの構成（フラッグシップ＝Leptos）
- [解説/rust1.85.md](./解説/rust1.85.md) … **Rust 言語そのもの**の解説（最新版 / edition 2024）。所有権・借用・ライフタイム・trait・async など。
- [leptos/](./leptos/) … **Leptos**（細粒度リアクティブ・SSR対応）。現状でRustフロントの注目株。

> ⚠️ Rustフロントは **0.x 系で API がよく変わる**。本リファレンスは概念理解が主目的。実装時は必ず最新の公式ドキュメントで API を確認すること。
> 関連: 土台は [../javascript/](../javascript/) / [../typescript/](../typescript/)（まずはこちらが実務の主流）。
