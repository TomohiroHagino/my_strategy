# Xcode

## ひとことで言うと
Appleが提供する**統合開発環境（IDE）**で、iOS / macOS / iPadOS / watchOS / visionOS アプリ開発の**必須ツール**。**Macでのみ動く**（Windows/Linux不可）。

## 役割・なぜ必要か
- Apple純正アプリ開発はXcodeに集約される。Swift/Objective-Cのコード編集から**ビルド・実行・デバッグ・署名・配布**まで、ひととおりを1つで完結できる。
- 主な機能:
  - コード編集（補完・リファクタ・SwiftのSyntax対応）
  - **シミュレータ / 実機デバッグ**（ブレークポイント、変数監視）
  - **SwiftUIライブプレビュー**（書いた画面を即座に確認）
  - 署名・配布（**App Store Connect** への提出）
  - **Instruments**（CPU / メモリ / リーク / 起動時間などのプロファイリング）
  - **SPM（Swift Package Manager）統合**（依存ライブラリ管理）
- 重要: **React Native / Flutter の iOS ビルドにも Xcode が必要**。CocoaPodsの解決やコード署名はXcode/付属ツールチェーン経由で行われるため、クロスプラットフォームでもMac+Xcodeから逃れられない。

## 基本の使い方
プロジェクトの構成要素:
- `.xcodeproj` … 単一プロジェクト
- `.xcworkspace` … 複数プロジェクト/Podsをまとめる入れ物（CocoaPods利用時はこちらを開く）
- **scheme** … ビルド対象＋実行設定の組み合わせ
- **target** … 成果物（アプリ本体・テスト・拡張など）の単位

よく使う操作:
```text
Cmd + B   ビルド
Cmd + R   ビルドして実行（シミュレータ/実機）
Cmd + .   実行停止
Cmd + U   テスト実行
```

CLIから扱う例:
```bash
# インストール済みSDK / シミュレータの確認
xcodebuild -showsdks
xcrun simctl list devices

# コマンドラインビルド（CIなどで使う）
xcodebuild -workspace MyApp.xcworkspace \
  -scheme MyApp -configuration Debug build

# RN/Flutterで使うCocoaPods
cd ios && pod install
```

## 実務での勘所
- CocoaPodsを使うプロジェクトは**必ず `.xcworkspace` を開く**（`.xcodeproj` だと依存が解決されずビルド失敗）。
- 署名は **Automatically manage signing** を有効にしておくと、開発中の証明書/プロファイル作成をXcodeが面倒を見てくれて楽。
- 実機テストには Apple Developer アカウント（無料でも可、ただし制約あり）と端末登録が要る。
- リリースビルドは **Product > Archive** から App Store Connect へアップロードする。
- パフォーマンス調査は **Instruments**。「なんとなく重い」を放置せず、リーク・起動時間を数値で確認する習慣をつける。
- CIではGUIを開かず `xcodebuild` を回す。schemeを「Shared」にしておかないとCIから見えない点に注意。

## ハマりどころ
- **Mac必須**: WindowsやLinuxではそもそも動かない。クラウドMac（CI）で代替するケースもある。
- **署名/証明書/provisioning地獄**: Certificate（証明書）・Provisioning Profile・App ID・Device登録の関係が崩れるとビルドが通らない。「Automatically manage signing」を切ったまま設定が古い、チームIDの不一致、期限切れなどが定番の詰まり。
- **Xcodeと iOS SDK のバージョン依存**: 新OS対応には新Xcodeが要り、逆に古いプロジェクトが新Xcodeで壊れることもある。OSバージョンとXcodeバージョンの対応表を意識する。
- **DerivedData のキャッシュ問題**: 原因不明のビルドエラーは `~/Library/Developer/Xcode/DerivedData` を消すと直ることがある。
  ```bash
  rm -rf ~/Library/Developer/Xcode/DerivedData
  ```
- **シミュレータと実機の差異**: シミュレータで動いても実機（メモリ制約・カメラ・プッシュ通知など）で落ちることがある。最終確認は実機で。
- RN/Flutter利用時、`pod install` 忘れや Pods のバージョン不整合で iOS だけビルドが落ちるのは頻出。

## 関連
[swiftui/swiftui6/getting_started.md](./swiftui/swiftui6/getting_started.md) / [README.md](./README.md)
