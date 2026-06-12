# パッケージ / pubspec.yaml（Flutter 3）

## ひとことで言うと
**`pubspec.yaml`** はプロジェクトの設定ファイルで、**依存パッケージ・アセット（画像等）・フォント** を宣言する。外部パッケージは [pub.dev](https://pub.dev) から取得し、`flutter pub get` で `pubspec.lock` 通りにインストールされる。

## 役割・なぜ必要か
- 「何に依存するか／何を同梱するか」を1ファイルに集約し、`flutter pub get` で誰の環境でも同じ依存をそろえる（再現性）。
- 車輪の再発明を避け、HTTP通信・状態管理・画像キャッシュ等は **pub.dev の実績あるパッケージ**を使うのが基本。
- 画像・フォントは「ただ置く」だけでは使えず、`pubspec.yaml` に登録して初めてビルドへ含まれる。

## 基本の書き方（コード）
```yaml
# pubspec.yaml（インデントは半角スペース2、タブ禁止）
name: my_app
description: "サンプルアプリ"
publish_to: "none"        # 社内/個人アプリは公開しない宣言

environment:
  sdk: ">=3.0.0 <4.0.0"   # Dart SDK の範囲

dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.0            # ^ は「1.2.0 以上 2.0.0 未満」を許す
  provider: ^6.1.0
  intl: ^0.19.0

dev_dependencies:         # 開発時のみ（テスト・lint・コード生成）
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0

flutter:
  uses-material-design: true

  assets:                 # 同梱する画像など（ファイル or ディレクトリ）
    - assets/images/logo.png
    - assets/images/      # 末尾 / でディレクトリ配下を一括登録

  fonts:                  # カスタムフォント
    - family: NotoSansJP
      fonts:
        - asset: assets/fonts/NotoSansJP-Regular.ttf
        - asset: assets/fonts/NotoSansJP-Bold.ttf
          weight: 700
```

```bash
# パッケージ追加（pubspec.yaml への追記まで自動）
flutter pub add http
flutter pub add --dev flutter_lints   # dev_dependencies に追加

# pubspec.yaml を手で編集した後は get で反映
flutter pub get

# 依存の更新（制約の範囲内で最新へ）
flutter pub upgrade

# どのバージョンが古いか確認
flutter pub outdated
```

```dart
// コードからの利用
import 'package:http/http.dart' as http;          // pub パッケージ
final logo = Image.asset('assets/images/logo.png'); // 登録済みアセット
const style = TextStyle(fontFamily: 'NotoSansJP');   // 登録済みフォント
```

## 実務での使い方・定番パターン
- **追加は `flutter pub add` を使う**：最新の妥当なバージョン制約を `pubspec.yaml` に自動で書いてくれる。手書きより事故が少ない。
- **`^1.2.0`（キャレット）**が標準。「メジャー据え置きでマイナー/パッチは追従」。固定したいなら `1.2.0`、範囲指定は `">=1.2.0 <2.0.0"`。
- **`pubspec.lock` はコミットする**（アプリの場合）。チーム・CIで実際に入る版を固定して再現性を確保。
- **`dependencies` と `dev_dependencies` を分ける**：テスト・lint・`build_runner` 等は dev 側へ。本番バンドルを軽く保つ。
- アセットはディレクトリ末尾 `/` で一括登録できる。命名は小文字・ハイフン推奨。
- 採用前に pub.dev で「likes / pub points / 最終更新 / null safety 対応 / プラットフォーム対応」を確認する。

## ハマりどころ / アンチパターン
- **YAML のインデント崩れ**：タブ混在や階層ずれで `pub get` が失敗、またはアセット未認識。**半角スペース2、タブ禁止**を徹底。リスト項目の `-` の位置に注意。
- **アセット未登録 / パス違い**：ファイルを置いただけでは読めない。`flutter:` 配下 `assets:` に正しい相対パスを書く。登録後は `flutter pub get`（場合によりホットリスタート／再ビルド）。
- **バージョン衝突（version solving failed）**：複数パッケージが同じ依存の別バージョンを要求して解決不能に。`flutter pub outdated` で原因を見て、制約を緩める／衝突相手を上げる／`dependency_overrides`（最終手段）で回避。
- **キャレットの誤解**：`^0.x` は「0.x 系のみ」で 0.(x+1) を含まない（1.0 系は別物）。0.x 台パッケージの更新は破壊的変更が来やすい。
- **`flutter pub get` し忘れ**：`pubspec.yaml` を編集したのに get していないと「Target of URI doesn't exist」等の import エラー。
- **不要・過剰な依存**：小機能のために重いパッケージを入れると保守とビルドが重くなる。まず標準/軽量で足りるか検討（YAGNI）。
- 取得が不安定な時は `flutter pub get` の前に `flutter clean`、それでも直らなければ `pubspec.lock` 削除→再 get を検討（チーム合意の上で）。

## 関連
[networking.md](./networking.md) / [getting_started.md](./getting_started.md) / [state_management.md](./state_management.md)
