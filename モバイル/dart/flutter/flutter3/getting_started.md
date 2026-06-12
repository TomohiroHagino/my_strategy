# 始め方（Getting Started）（Flutter 3）

## ひとことで言うと
プロジェクトを作り（`flutter create`）→ 実機/エミュレータで動かし（`flutter run`）→ コードを直せば即反映（**ホットリロード**）。これが Flutter 開発を始める"最初の一周"。

## 役割・なぜ必要か
- Flutter は「**1つの Dart コードから iOS / Android / Web / デスクトップを動かす**」ためのツールキット。最初に環境（SDK・デバイス）を整え、雛形を生成して動かすところがスタート地点。
- 開発速度の核は **ホットリロード**：保存するだけで UI が一瞬で書き換わり、画面の状態を保ったまま見た目を確認できる。これが「試行錯誤を速く回せる」最大の理由。
- まず `flutter doctor` で環境の欠けを潰しておかないと、後で「ビルドできない」「デバイスが見えない」で詰まる。最初の投資。

## 基本の書き方（コード）
```bash
# 0) 環境チェック（最初に必ず）。✗ の項目を1つずつ潰す
flutter doctor

# 1) アプリ雛形を作る（パッケージ名は逆ドメインで指定すると堅い）
flutter create --org com.example my_app
cd my_app

# 2) 接続中のデバイス/エミュレータを確認
flutter devices

# 3) 起動（デバイスが複数なら -d で指定）
flutter run                 # 自動選択
flutter run -d chrome       # Web で起動
flutter run -d emulator-5554

# 4) 起動後、ターミナルで使うキー
#   r → ホットリロード（状態を保ったまま反映）
#   R → ホットリスタート（状態を捨ててアプリ再起動）
#   q → 終了
```

```dart
// lib/main.dart … エントリポイント
import 'package:flutter/material.dart';

// main() がアプリの開始点。runApp() に「ルートの Widget」を渡す
void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    // MaterialApp = Material Design アプリの土台（テーマ・ルーティング等を束ねる）
    return MaterialApp(
      title: 'My App',
      theme: ThemeData(colorSchemeSeed: Colors.blue),
      home: const HomePage(),
    );
  }
}

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    // Scaffold = 1画面の骨組み（AppBar / body / FAB などの定位置を用意）
    return Scaffold(
      appBar: AppBar(title: const Text('ホーム')),
      body: const Center(child: Text('Hello, Flutter 3')),
    );
  }
}
```

## 実務での使い方・定番パターン
- **`flutter create` の直後にやること**：`--org` でパッケージ名を決め、不要なサンプル（カウンター）を消して自分の画面に置き換える。後からのリネームは面倒なので最初に決める。
- **ホットリロードとリスタートの使い分け**：見た目の調整は `r`（速い・状態維持）。`initState`・`main()`・グローバル変数の初期化・`enum`/定数の変更などは反映されないので `R`（フルリスタート）で確認する。
- **起動構成**：VS Code / Android Studio から実行すると `flutter run` 相当 + デバッガが付く。`flutter run --release` で本番に近い速度を確認できる（リロードは効かない）。
- **デバイス確保**：iOS は Mac + Xcode、Android はエミュレータ（AVD）か実機（USB デバッグ ON）。Web/デスクトップは追加設定が要る場合がある。

## ハマりどころ / アンチパターン
- **`flutter doctor` の ✗ を放置**：Android licenses 未同意（`flutter doctor --android-licenses`）、Xcode 未セットアップ等が残ると本ビルドで詰まる。最初に全部 ✓ にする。
- **「リロードしたのに反映されない」**：それは `r` で拾えない変更（`main()`・`initState`・定数・依存追加など）。**迷ったら `R`（ホットリスタート）**、それでもダメなら一旦停止して `flutter run` し直す。
- **デバイスが見えない / 選ばれない**：`flutter devices` で確認。複数あるのに `-d` を付けず固まる、エミュレータ未起動、USB デバッグ OFF が定番原因。
- **`pubspec.yaml` を編集したのにリロード**：依存追加は `flutter pub get`＋フルリスタートが要る（`r` では入らない）。
- **`const` を付け忘れた Widget**：動くが再ビルドが無駄に増える。雛形の段階から `const` を意識する。→ [widgets.md](./widgets.md)

## フォルダ構成（始動直後）
```
myapp/
├── lib/
│   └── main.dart           # アプリの起点（Dartコードはここ）
├── test/
│   └── widget_test.dart    # サンプルのウィジェットテスト
├── android/                # Android プロジェクト
│   ├── app/                #   アプリモジュール
│   └── build.gradle        #   ビルド設定
├── ios/                    # iOS プロジェクト
│   ├── Runner/             #   アプリ本体
│   └── Runner.xcodeproj    #   Xcode プロジェクト
├── web/
│   └── index.html          # Web のエントリHTML
├── macos/  linux/  windows/  # 各デスクトップOS
├── pubspec.yaml            # 依存・アセット宣言
├── pubspec.lock            # 依存の固定（自動生成）
├── analysis_options.yaml   # 静的解析（lint）設定
├── .metadata               # Flutter が管理するメタ情報
├── .gitignore              # build/ 等を除外
└── myapp.iml               # IDE プロジェクトファイル
```
- Dartコードは基本 `lib/` だけ。各OS固有設定は `android/` `ios/` 等に。

## 関連
[widgets.md](./widgets.md) / [layout.md](./layout.md)
