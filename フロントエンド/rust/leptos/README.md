# Leptos

## 一言で
Rust 製の**細粒度リアクティブ**Webフレームワーク。**signals（シグナル）**で「変わった値に依存する所だけ」を更新する（仮想DOM diffをしない）。SolidJS に近い設計で、**CSR も SSR/フルスタックも**書ける。`view!` マクロでJSXライクにUIを記述。

## 特徴
- **signals による細粒度リアクティビティ**：`signal()` で作り、`.get()`/`.set()`。依存する箇所だけピンポイント更新。
- **`#[component]` ＋ `view!` マクロ**：関数コンポーネント＋JSXライクなテンプレート。
- **`#[server]`（サーバ関数）**：クライアントから呼ぶとサーバで実行（フルスタック・型共有）。
- **SSR / ハイドレーション**：`cargo-leptos` でSSRビルド。Axum/Actix と統合。

## このフォルダの構成
- [leptos0/](./leptos0/) … **Leptos 0.x 実務リファレンス（フラッグシップ）**。始め方〜signals〜view!〜サーバ関数〜SSR〜ルーティング〜罠まで、項目=1ファイル。
  - ※ Leptos は 0.x 系。フォルダ名 `leptos0` はメジャー番号0（rails7 等と同じ命名規則）。**API変化が速いので最新docs要確認**。

> 関連: Rustフロントの全体像は [../README.md](../README.md)。
