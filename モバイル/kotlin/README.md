# Kotlin

## 一言で
JetBrains の**静的型付け・null安全**言語。**Android 開発の公式言語**で、Java と完全相互運用しつつ簡潔・安全に書ける。現在のAndroid UIは **Jetpack Compose**（宣言的）で書く。サーバ(Ktor/Spring)やマルチプラットフォーム(KMP)にも使える。

## 特徴
- **null安全**（`String?` と `String` を型で区別）。
- **簡潔**：`data class`・`when`・拡張関数・スマートキャスト。
- **コルーチン**：`suspend` 関数による軽量な非同期/並行処理。
- **Java と完全相互運用**（既存資産を活かせる）。
- **Jetpack Compose**（宣言的UI）/ 旧来は XML レイアウト。

## どういう使い方をするのか
- **Android ネイティブアプリ**（公式言語）。
- サーバ（Ktor / Spring Boot）、**Kotlin Multiplatform（KMP）** で iOS/Android 共通ロジック。

## 強み / 弱み
- 強み：null安全・簡潔・コルーチン・Java資産活用・Android公式。
- 弱み：Android外では用途が限定的・**開発に Android Studio＋Gradle** が要る（ビルドが重め）。

## エコシステム・周辺
- IDE: **Android Studio**（→ [android_studio.md](./android_studio.md)）
- UI: **Jetpack Compose**（→ [compose/](./compose/)）/ XML
- 非同期: Coroutines / Flow、DI: Hilt、ビルド: Gradle

## このフォルダの構成
- [解説/kotlin2.1.md](./解説/kotlin2.1.md) … **Kotlin 言語そのもの**の解説（最新版）。文法・null安全・コルーチン・K2 コンパイラなど。
- [compose/](./compose/) … **Jetpack Compose**（現行の宣言的UIフレームワーク）のフラッグシップリファレンス。
- [android_studio.md](./android_studio.md) … Android Studio（Kotlinネイティブ開発のIDE）とは。
