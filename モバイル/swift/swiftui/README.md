# SwiftUI

## 一言で
Apple の**宣言的UIフレームワーク**。「**状態(State)から画面を宣言**し、状態が変われば自動で再描画」する（React/Flutter と同じ発想）。`View` は `struct`で、`body` にUIを記述。Xcode の**ライブプレビュー**で即確認できる。

## 特徴
- **宣言的・状態駆動**：`@State` 等が変わると関連ビューだけ再描画。
- **View は値型（struct）**：軽量、合成して組む。
- **modifier チェーン**：`.padding().background(...)` で見た目を重ねる。
- **Xcode プレビュー**で即時確認。
- iOS 13+ で登場、iOS 17 で `@Observable` 等が刷新。

## このフォルダの構成
- [swiftui6/](./swiftui6/) … **SwiftUI 実務リファレンス（フラッグシップ）**。始め方〜View〜状態〜ナビゲーション〜非同期〜罠まで、項目=1ファイル。
  - ※ フォルダ名 `swiftui6` は Swift 6 / iOS 18 era の SwiftUI を表す（API は iOS バージョンに追随）。

> 関連: 言語は [../README.md](../README.md)（Swift）、IDE は [../xcode.md](../xcode.md)（Xcode）。
