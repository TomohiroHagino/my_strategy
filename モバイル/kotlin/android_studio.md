# Android Studio（Kotlin / ネイティブAndroid用）

## ひとことで言うと
Googleが提供する**Android開発の公式IDE**（JetBrainsの**IntelliJ IDEA**がベース）。Kotlin/Javaによる**ネイティブAndroid開発の標準環境**。

## 役割・なぜ必要か
- Androidアプリ開発に必要なものが一式そろう。コードを書くだけでなく、ビルド・エミュレータ実行・デバッグ・成果物生成までを統合的に扱える。
- 主な機能:
  - コード編集（Kotlin補完・リファクタ・Lint）
  - **Gradleビルド**（依存解決・ビルド構成の中心）
  - **AVD（エミュレータ）**による仮想端末での実行
  - **Composeプレビュー**（Jetpack Composeの画面を即確認）
  - **Logcat**（端末/エミュレータのログ閲覧・フィルタ）
  - デバッガ / プロファイラ（CPU・メモリ・ネットワーク）
  - **APK / AAB（Android App Bundle）生成**（配布用成果物）
- IntelliJベースなので、補完やリファクタの質が高く、Kotlinとの相性が良いのが採用理由。

## 基本の使い方
プロジェクトの主要素:
- `build.gradle.kts`（または `build.gradle`）… ビルド設定・依存定義。**Kotlin DSL（.kts）**が現在の主流。
- `settings.gradle.kts` … モジュール構成
- **SDK Manager** … Android SDK / Platform / Build Tools の導入・更新
- **AVD Manager** … エミュレータ端末の作成・管理

よく使う操作:
```text
Shift + F10   Run（実行）
Shift + F9    Debug（デバッグ実行）
Ctrl + F9     Build（ビルド）
```

依存定義の例:
```kotlin
// build.gradle.kts（モジュール）
dependencies {
    implementation("androidx.core:core-ktx:1.13.0")
    implementation(platform("androidx.compose:compose-bom:2024.05.00"))
    implementation("androidx.compose.ui:ui")
}
```

CLIからも扱える:
```bash
./gradlew assembleDebug   # Debug APKをビルド
./gradlew bundleRelease   # Release AABを生成
./gradlew test            # ユニットテスト
```

## 実務での勘所
- **keystore署名**: リリースAABには署名が必須。keystoreファイルとパスワードは**安全に管理**し、Gitに含めない（環境変数や`local.properties`/CIシークレットで渡す）。
- **build variants**: `debug` / `release` や `free` / `paid` などのフレーバーをビルドタイプ×プロダクトフレーバーで切り替える。APIエンドポイントの出し分けに便利。
- **R8**（旧ProGuard）: リリースビルドでコード縮小・難読化を行う。リフレクション利用箇所は `proguard-rules.pro` で**keepルール**を書かないと実行時に落ちる。
- 配布は AAB（`.aab`）が標準。APKは検証/サイドロード向け。
- Composeプレビューを活用すると、エミュレータ起動を待たずにUIを高速に確認できる。

## ハマりどころ
- **Gradle同期が重い / 失敗する**: 初回や依存追加時に時間がかかる。失敗時は以下が定番。
  ```bash
  ./gradlew --stop          # デーモン停止
  ./gradlew clean build --refresh-dependencies
  ```
  キャッシュ起因なら `~/.gradle/caches` の削除も検討。
- **SDK / JDKバージョン不一致**: `compileSdk` / `targetSdk` と インストール済みSDK、Gradle Pluginが要求するJDKがズレるとビルドが通らない。エラーメッセージのバージョン要求を素直に合わせる。
- **AVDのリソース消費**: エミュレータはCPU/メモリを大きく食う。実機デバッグ（USB / `adb`）の方が軽快なことも多い。
- **メモリ不足**: 大きめプロジェクトでは Gradle / IDE がメモリを圧迫。`gradle.properties` の `org.gradle.jvmargs=-Xmx4g` などで調整する。
- Kotlin / AGP（Android Gradle Plugin）/ Gradle の三者バージョン互換に注意。片方だけ上げると壊れることがある。

## 関連
[compose/compose1/getting_started.md](./compose/compose1/getting_started.md) / [README.md](./README.md)
