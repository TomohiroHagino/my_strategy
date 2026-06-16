# Android Studio（React Native用）

## ひとことで言うと
React Native開発において、**メインのコード編集IDEではない**（JS/TSは VS Code 等で書く）が、**Android側のSDK提供・エミュレータ・ネイティブビルド/デバッグ**のために使うツール。

## 役割・なぜ必要か
- RNのアプリ本体（画面ロジック）はJavaScript/TypeScriptで書くため、普段のコーディングはVS Codeなど軽量エディタで十分。Android Studioは**Android基盤を用意する役**として必要になる。
- RNでAndroid Studioが担うこと:
  - **Android SDK / `platform-tools`（`adb`等）/ NDK** の提供
  - **AVD（エミュレータ）** の作成・起動
  - `android/` ディレクトリ（ネイティブAndroidプロジェクト）の**Gradleビルド**
  - **Logcat** によるネイティブ層のログ確認（JSのconsole.logでは追えないクラッシュの調査）
- `ANDROID_HOME` 環境変数で SDK の場所をツールチェーンに知らせる。これが未設定だと `npx react-native run-android` 等が SDK を見つけられない。

## 基本の使い方
環境変数の設定例:
```bash
# ~/.zshrc / ~/.bashrc
export ANDROID_HOME=$HOME/Library/Android/sdk   # macOS例
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/emulator
```

普段の開発フロー（多くはCLI/Metro主体）:
```bash
# 接続中の端末/エミュレータ確認
adb devices

# Android実行（裏でGradleビルド → アプリ起動）
npx react-native run-android
# Expo利用時
npx expo run:android
```

Android Studioを実際に開く場面:
- AVD Manager でエミュレータを作る/起動する
- `android/` を開いてネイティブビルドエラーを調べる
- Logcat でネイティブクラッシュのスタックトレースを読む

## 実務での勘所
- **普段は `npx expo` / Metro で開発**し、Android Studioは「**SDK・エミュレータの提供**」と「**ネイティブ問題のデバッグ**」のときに開く、という使い分けが基本。常時起動しておく必要はない。
- **iOSはXcode**が担当（Macが必要）。AndroidをAS、iOSをXcode、という二本立てになる。
- ネイティブモジュールを追加・自作したときは、AS側のGradleビルドやLogcatが必須の調査ツールになる。→ [native_modules.md](./native_modules.md)
- **Expo Goのみで完結する開発なら**、Android Studioの出番は最小限（エミュレータが欲しいときくらい）。実機にExpo Goを入れれば不要にもできる。

## ハマりどころ
- **`ANDROID_HOME` / SDKパス未設定**: 最頻出。`run-android` が SDK を見つけられずエラー。`adb devices` が通るかでまず切り分ける。
- **Gradleバージョン不一致**: RNのバージョンが要求するGradle/AGPと、ローカルのバージョンがズレると `android/` のビルドが落ちる。RNアップグレード時は `android/gradle/wrapper/gradle-wrapper.properties` の確認を。
- **エミュレータ設定**: AVD未作成だと実行先がない。ハードウェアアクセラレーション（HAXM/KVM）未設定だと激重。
- **JS書くのにASは不要**: 画面ロジックの編集にAndroid Studioを開く必要はない（重いだけ）。VS Code等で書き、ASはAndroid基盤用に限定する。
- SDK License未同意でビルドが止まることがある。`sdkmanager --licenses` で同意する。

## 関連
[getting_started.md](./getting_started.md) / [native_modules.md](./native_modules.md)
