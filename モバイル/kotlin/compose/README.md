# Jetpack Compose

## 一言で
Android の**宣言的UIフレームワーク**（Kotlin製）。`@Composable` 関数でUIを記述し、「**状態が変われば再コンポーズ（再描画）**」する（SwiftUI/React と同じ発想）。旧来の XML レイアウトに代わる現行の標準。

## 特徴
- **`@Composable` 関数でUI**：UIを関数として組む。
- **状態駆動・再コンポーズ**：`remember { mutableStateOf(...) }` が変わると再描画。
- **`Modifier` チェーン**：`Modifier.padding().background(...)` で見た目を重ねる。
- **Material 3** 既製デザイン、Android Studio のプレビュー。
- コルーチンと統合（`LaunchedEffect` 等）。

## このフォルダの構成
- [compose1/](./compose1/) … **Jetpack Compose 実務リファレンス（フラッグシップ）**。始め方〜Composable〜状態〜ナビゲーション〜ViewModel〜コルーチン〜罠まで、項目=1ファイル。
  - ※ フォルダ名 `compose1` は Compose 1.x を表す。

> 関連: 言語は [../README.md](../README.md)（Kotlin）、IDE は [../android_studio.md](../android_studio.md)（Android Studio）。
