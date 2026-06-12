# スタイル / テーマ（Material / Cupertino）（Flutter 3）

## ひとことで言うと
アプリ全体の見た目（色・文字・余白）を**一元管理する仕組み**。`MaterialApp` の `theme` に **`ThemeData`** を渡し、各Widgetは **`Theme.of(context)`** で参照する。デザイン体系は Android風の **Material** と iOS風の **Cupertino** の2系統がある。

## 役割・なぜ必要か
- 色やフォントを各Widgetに**ベタ書きすると、変更時に全箇所を直す羽目**になる。テーマに集約すれば1か所変えれば全画面に反映される（DRY）。
- ブランドカラー・ダークモード対応・一貫したUIを**保証する基盤**。デザインの「真実の源」になる。
- `Material`（`MaterialApp`）と `Cupertino`（`CupertinoApp`）でデフォルトの見た目・挙動が違うため、**どちらの世界で組むかを決める**こと自体が設計判断になる。

## 基本の書き方（コード）
```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // ライトテーマ（seed色から自動でカラーパレット生成）
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
        textTheme: const TextTheme(
          titleLarge: TextStyle(fontSize: 22, fontWeight: FontWeight.bold),
        ),
      ),
      // ダークテーマ
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: Colors.indigo,
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      themeMode: ThemeMode.system, // 端末設定に追従（light/dark/system）
      home: const HomePage(),
    );
  }
}

class HomePage extends StatelessWidget {
  const HomePage({super.key});
  @override
  Widget build(BuildContext context) {
    // ★テーマを参照（色・文字をハードコードしない）
    final theme = Theme.of(context);
    return Scaffold(
      appBar: AppBar(title: const Text('テーマ例')),
      body: Padding(
        padding: const EdgeInsets.all(16), // 余白も統一して扱う
        child: Text(
          'こんにちは',
          style: theme.textTheme.titleLarge?.copyWith(
            color: theme.colorScheme.primary,
          ),
        ),
      ),
    );
  }
}
```

```dart
// Cupertino（iOS風）の最小例
import 'package:flutter/cupertino.dart';

class IosApp extends StatelessWidget {
  const IosApp({super.key});
  @override
  Widget build(BuildContext context) {
    return const CupertinoApp(
      theme: CupertinoThemeData(primaryColor: CupertinoColors.activeBlue),
      home: CupertinoPageScaffold(
        navigationBar: CupertinoNavigationBar(middle: Text('iOS風')),
        child: Center(child: Text('Cupertino')),
      ),
    );
  }
}
```

## 実務での使い方・定番パターン
- **色は `Theme.of(context).colorScheme` から**取る：`primary` / `secondary` / `surface` / `error` / `onPrimary`（＝primary上の文字色）など意味付きで使うと、ダークモードでも自動で破綻しない。
- **文字は `textTheme` から**：`bodyMedium` / `titleLarge` 等の役割名で使い、個別 `TextStyle` は `copyWith` で部分上書き。
- **余白は `EdgeInsets`** で統一。`EdgeInsets.all(16)` / `.symmetric(horizontal: 16, vertical: 8)` / `.only(top: 8)`。数値は `8の倍数` など基準を決めると整う。
- **ダークモード**：`theme` と `darkTheme` を両方定義し `themeMode` で切替。手動切替は状態管理で `ThemeMode` を持つ。→ [state_management.md](./state_management.md)
- **コンポーネント単位のテーマ**：`ThemeData` 内の `elevatedButtonTheme` / `appBarTheme` / `cardTheme` 等で部品ごとの既定を一括設定できる。
- **どちらの体系か決める**：Android/iOS両対応で一貫させたいなら Material 1本が無難。OSネイティブ感を出したい時だけ `Platform.isIOS` で Cupertino を出し分ける。

## ハマりどころ / アンチパターン
- **色・サイズをハードコード**：`Color(0xFF3F51B5)` や `fontSize: 16` を各所にベタ書きすると、ダークモード化やデザイン変更で全滅。必ずテーマから引く。
- **`Theme.of(context)` の `context` が間違い**：`ThemeData` を定義した `MaterialApp` より**上の context**で `Theme.of` するとデフォルト値が返る。テーマを使う側は必ず `MaterialApp` 配下のWidgetで呼ぶ（`Builder` でcontextを掘り直すことも）。
- **Material と Cupertino の混在**：`CupertinoApp` 配下で Material 専用Widget（`Scaffold` 等）を使うと `Material` ウィジェットが無いエラー。逆も同様。世界を跨ぐWidgetは慎重に。
- **`useMaterial3` の有無で見た目が変わる**：M2とM3で色の出方・部品形状が違う。新規は `useMaterial3: true` 前提で統一する。
- **`colorScheme` を使わず `primaryColor` 等の旧APIに頼る**：M3では多くが非推奨。`ColorScheme.fromSeed` を起点にする。
- **ダークモードで文字が読めない**：背景に固定色、文字に `colorScheme.onSurface` 等を使い分けないとコントラスト崩壊。`on○○` 系の色を使う。
- **`EdgeInsets` を使わず生の `Padding` を多重ネスト**して余白がバラバラ → 余白の基準値を決めて統一する。

## 関連
[layout.md](./layout.md) / [widgets.md](./widgets.md) / [state_management.md](./state_management.md)
