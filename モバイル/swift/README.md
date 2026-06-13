# Swift

## 一言で
Apple の**静的型付け・安全志向**の言語。iOS / macOS / watchOS / visionOS ネイティブ開発の標準。**Optional（null安全）**・値型中心・プロトコル指向が特徴で、現在のUIは **SwiftUI**（宣言的）で書く（旧来は UIKit）。

## 特徴
- **Optional（`String?`）で null を型で扱う**＝安全。
- **値型中心**（`struct`/`enum`）＋ **プロトコル指向**。
- **ARC**（自動参照カウント）でメモリ管理。
- **`async`/`await`・actor** による並行処理。
- **SwiftUI**（宣言的UI）/ UIKit（命令的・旧来）。

## どういう使い方をするのか
- **iOS / macOS / visionOS のネイティブアプリ**（App Store）。
- Apple純正の性能・OS機能をフルに使いたいとき。

## 強み / 弱み
- 強み：型安全・高性能・Apple機能を最大限活用・SwiftUIで生産的。
- 弱み：**Apple プラットフォーム中心**（クロスではない）・開発に **Mac＋Xcode 必須**。

## エコシステム・周辺
- IDE: **Xcode**（→ [xcode.md](./xcode.md)）
- UI: **SwiftUI**（→ [swiftui/](./swiftui/)）/ UIKit
- 依存管理: Swift Package Manager (SPM) / CocoaPods

## このフォルダの構成
- [解説/swift6.md](./解説/swift6.md) … **Swift 言語そのもの**の解説（最新版）。値型・Optional・プロトコル指向・actor/data race safety など。
- [swiftui/](./swiftui/) … **SwiftUI**（現行の宣言的UIフレームワーク）のフラッグシップリファレンス。
- [xcode.md](./xcode.md) … Apple の IDE「Xcode」とは（iOS開発の必須ツール）。
