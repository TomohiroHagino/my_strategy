# 始め方（React Native）

## ひとことで言うと
React Native（RN）プロジェクトを**作って・起動して・端末に映す**までの一連の流れ。いまの主流入口は **Expo**（`npx create-expo-app`）で、ネイティブ環境構築を肩代わりしてくれる。

## 役割・なぜ必要か
- RNは「JS/TSで書いたUIをネイティブ（iOS/Android）として動かす」ため、**Metro（JSバンドラ）＋ネイティブビルド**という2層構造を持つ。ここを最初に理解しておかないと、後で詰まる。
- 入口は大きく2つ:
  - **Expo**（推奨・主流）… ネイティブ設定を抽象化。最短で起動でき、後述の **dev build** でネイティブも扱える。0.7x の **New Architecture** にも追従。
  - **Bare React Native CLI**（`npx @react-native-community/cli init`）… `ios/` `android/` を最初から自前管理。完全な自由度が要るとき。
- 迷ったら **Expo 一択**でよい。Expo は途中で `npx expo prebuild` して bare 相当に「展開」もできるので、後戻りもきく。

## 基本の書き方（コード）
```bash
# 1) プロジェクト作成（TypeScript テンプレートが既定）
npx create-expo-app@latest my-app
cd my-app

# 2) 開発サーバ（Metro）起動。QR・操作メニューが出る
npx expo start
#   i → iOS シミュレータ / a → Android エミュレータ / w → Web
#   r → リロード / j → デバッガ / m → 開発メニュー

# 3) 実機で見る: Expo Go アプリを入れて QR を読む（最速）
#    ※ Expo Go は「Expoが内包する標準ネイティブ」だけで動く
```

```bash
# ネイティブの追加ライブラリ（カメラ等）を使うなら dev build を作る
npx expo install expo-dev-client
npx expo run:ios       # Xcode でビルドして実機/シミュレータへ
npx expo run:android   # Android Studio のツールチェーンでビルド
# 以後は `npx expo start --dev-client` で自前ビルドのアプリへ接続
```

```bash
# Bare CLI で始める場合（ネイティブを自前管理したいとき）
npx @react-native-community/cli@latest init MyApp
cd MyApp && npm run ios   # or npm run android
```

## 実務での使い方・定番パターン
- **Expo Go で素早く試作 → ネイティブ依存が出たら dev build へ移行**、が王道。Expo Go は「お試し環境」、dev build は「自分のアプリ専用クライアント」と捉える。
- **`expo install` を使う**（`npm install` ではなく）。Expo SDK バージョンに合うライブラリ版を解決してくれるので相性事故が減る。
- **シミュレータ/エミュレータ**: iOS は **Xcode**（macOS必須）、Android は **Android Studio** の AVD。実機は USB 接続 or 同一 Wi-Fi で Expo Go。
- **New Architecture（Fabric/TurboModules）** は Expo SDK 51+ で既定有効化が進む。新規なら最初から ON で問題ないことが多い。
- **EAS Build**（`eas build`）でクラウドビルド＆配布。ローカルに Xcode が無くても iOS ビルドを作れる。

## ハマりどころ / アンチパターン
- **「Expo か Bare CLI か」を決めずに始める**。情報がどちらのものか分からず混乱する。まず Expo に寄せるのが安全。
- **Expo Go で動かないライブラリを入れて『動かない！』**。カメラ・BLE 等ネイティブコードを含むものは Expo Go 不可 → **dev build が必要**。エラー文の "requires a development build" がサイン。
- **ネイティブ環境を入れずに `run:ios`／`run:android`**。Xcode / Android Studio（SDK・エミュレータ・JDK）が未整備だとビルドが落ちる。最初の関門はここ。
- **Metro のキャッシュ起因の不可解なエラー**。`npx expo start -c`（キャッシュクリア）で直ることが多い。
- **`npm install` でバージョン不整合**。Expo 管理下では原則 `expo install` を使う。
- **Windows で iOS を動かそうとする**。iOS ビルドは macOS（か EAS のクラウド）が必要。

## フォルダ構成（始動直後）
```
MyApp/                         # 素の React Native CLI の場合
├── App.tsx                    # 画面の起点
├── index.js                   # AppRegistry への登録
├── android/                   # Android プロジェクト
│   ├── app/                   #   アプリモジュール
│   └── build.gradle           #   ビルド設定
├── ios/                       # iOS プロジェクト
│   ├── Podfile                #   CocoaPods 依存
│   └── MyApp.xcodeproj        #   Xcode プロジェクト
├── __tests__/                 # テスト
├── package.json               # 依存・scripts
├── tsconfig.json              # TS設定
├── babel.config.js            # Babel 設定
├── metro.config.js            # Metro（JSバンドラ）設定
├── app.json                   # アプリ名等
├── .watchmanconfig            # Watchman 設定
└── Gemfile                    # CocoaPods 用 Ruby 依存
```
```
MyApp/                         # Expo の場合
├── app/                       # expo-router（ファイルベースルーティング）
├── app.json                   # Expo 設定
├── assets/                    # 画像・フォント
└── （android/ ios/ は隠蔽。必要時に prebuild で展開）
```
- 素のRNは `android/` `ios/` が見える。Expo はそれらを隠して `app/` 中心。

## 関連
[core_components.md](./core_components.md) / [native_modules.md](./native_modules.md)
