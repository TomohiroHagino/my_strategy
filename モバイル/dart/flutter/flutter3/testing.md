# テスト（Testing）（Flutter 3）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。Flutterは標準で3層を提供する：**unit test**（純粋なロジック）、**widget test**（Widget単体をメモリ上で描画して検証）、**integration test**（実機/シミュレータで全体を動かすE2E）。`flutter test` で一括実行できる。

## 役割・なぜ必要か
- 変更のたびに手で全画面を確認するのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- widget test は実機不要・高速（ヘッドレス描画）で、UIロジックを大量に守れる。
- 「この仕様で動く」という実行可能な仕様書になり、リファクタの安全網になる。

## 基本の書き方（コード）
```dart
// test/calc_test.dart（unit test：純粋なロジック。package:test）
import 'package:test/test.dart';
import 'package:myapp/calc.dart';

void main() {
  group('add', () {
    test('1 + 2 は 3', () {
      expect(add(1, 2), 3);          // expect(実際, 期待)
    });
    test('負数も扱える', () {
      expect(add(-1, -2), -3);
    });
  });
}
```
```dart
// test/counter_widget_test.dart（widget test：testWidgets + WidgetTester）
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:myapp/counter_page.dart';

void main() {
  testWidgets('+ボタンでカウントが増える', (WidgetTester tester) async {
    // 1) Widgetをメモリ上に描画（MaterialAppで包むのが定番）
    await tester.pumpWidget(const MaterialApp(home: CounterPage()));

    // 2) find で要素を特定 → 初期状態を検証
    expect(find.text('0'), findsOneWidget);
    expect(find.text('1'), findsNothing);

    // 3) 操作（タップ）→ pump で1フレーム再描画
    await tester.tap(find.byIcon(Icons.add));
    await tester.pump();

    // 4) 結果を検証
    expect(find.text('1'), findsOneWidget);
  });

  testWidgets('非同期完了を待つ', (tester) async {
    await tester.pumpWidget(const MaterialApp(home: CounterPage()));
    await tester.tap(find.text('Load'));
    // アニメ/遅延が落ち着くまで繰り返しpump（タイマーが残ると例外）
    await tester.pumpAndSettle();
    expect(find.text('Done'), findsOneWidget);
  });
}
```
```dart
// integration_test/app_test.dart（integration test：実機/シミュレータでE2E）
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:myapp/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  testWidgets('起動して画面遷移できる', (tester) async {
    app.main();
    await tester.pumpAndSettle();
    await tester.tap(find.byKey(const Key('next')));
    await tester.pumpAndSettle();
    expect(find.text('Detail'), findsOneWidget);
  });
}
// 実行: flutter test integration_test/app_test.dart
```
```dart
// test/golden_test.dart（golden test：描画結果を画像で固定し差分検出）
testWidgets('ボタンの見た目が崩れていない', (tester) async {
  await tester.pumpWidget(const MaterialApp(home: MyButton()));
  await expectLater(
    find.byType(MyButton),
    matchesGoldenFile('goldens/my_button.png'), // 初回生成: flutter test --update-goldens
  );
});
```

## 実務での使い方・定番パターン
- **テストピラミッド**：土台に多数の高速な **unit test**、中間に **widget test**（実機不要でUIを守る主役）、頂点に少数の **integration test**（重要フローのE2Eだけ）。
- `pubspec.yaml` の `dev_dependencies` に `flutter_test`（SDK同梱）、E2Eは `integration_test` を追加。純ロジック用に `test` も。
- **`find` の使い分け**：`find.text('OK')` / `find.byType(ElevatedButton)` / `find.byIcon(Icons.add)` / `find.byKey(Key('id'))`。安定狙いなら `Key` 指定が堅い。
- **検証マッチャ**：`findsOneWidget` / `findsNothing` / `findsNWidgets(n)` / `findsWidgets`。
- 状態管理（Provider/Riverpod）は**テスト時にfake/モックへ差し替え**て注入し、ネットワーク等の外部依存を切る。
- **golden test** はボタン・カードなど見た目重要部品に。フォント差で環境依存になりやすいので CI で生成基準を統一する。
- 操作後は必ず再描画：1フレームだけなら `pump()`、アニメ/遅延が絡むなら `pumpAndSettle()`。

## ハマりどころ / アンチパターン
- **`pumpWidget` 前に `find` / `tap`**：まだ何も描画されておらず必ず失敗。最初に `await tester.pumpWidget(...)`。
- **`MaterialApp` で包み忘れ**：`Theme.of` / `Navigator` / `Directionality` が無くて例外。テストでも `MaterialApp(home: ...)` で包む。
- **`pump()` と `pumpAndSettle()` の取り違え**：`pump()` は1フレームだけ→非同期/アニメが未完で検証が空振り。逆に**無限アニメ**に `pumpAndSettle()` を使うとタイムアウトで永久に止まる→`pump(Duration(...))` で刻む。
- **`find.text` がマッチしない**：表記揺れ（全角/半角・前後空白）や、`RichText`/複数要素で `findsOneWidget` が落ちる。`findsWidgets` や `Key` 指定に切り替える。
- **非同期の待ち忘れ**：`tap` 直後に検証して未反映。`Future`/タイマーが残ると "Pending timers" 例外。`await tester.pump()` か `pumpAndSettle()` で確実に待つ。
- **`find.byType` の曖昧さ**：同型Widgetが複数あると特定できない。`find.byKey` で一意化。
- **golden の環境差**：OS/フォントレンダリング差で差分が出る。CI環境で `--update-goldens` を回し基準画像を固定。
- カバレッジ偏重：数字（80%目安）だけ追って**重要フローの integration test が無い**のは本末転倒。`flutter test --coverage` で計測しつつ要所を厚く。

## 関連
[widgets.md](./widgets.md) / [pitfalls.md](./pitfalls.md)
