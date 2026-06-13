# Dart 3.5（言語解説）

## ひとことで言うと
静的型付け・健全な null safety を持ち、JIT（開発時のホットリロード）と AOT（本番のネイティブバイナリ）を切り替えて使うクライアント最適化言語。3.x 系では records・patterns・sealed class が加わり、型でデータの形を表して網羅的に分岐できるようになった。

## このバージョンの位置づけ（リリース / サポート / どこで使うか）
- Dart 3.5 は 2024 年中盤の安定版で、Flutter 3.24 と同時に配布された。Dart のバージョンは Flutter SDK に同梱されており、`flutter --version` で確認できる。
- 言語仕様としての大きな転換点は 3.0（records / patterns / class modifiers の導入、null safety の完全必須化）であり、3.5 はそれを土台にした安定的な改良リリース。
- 主な用途は Flutter アプリ（モバイル / Web / Desktop）。CLI やサーバ（`dart:io`、shelf）も書けるが主流ではない。

```bash
dart --version          # Dart SDK のバージョン確認
dart create my_app      # CLI プロジェクト雛形
dart run                # JIT で実行
dart compile exe bin/main.dart   # AOT でネイティブ実行ファイル生成
```

## 言語の基本（文法の要点）
変数は `var`（型推論）/ `final`（再代入不可）/ `const`（コンパイル時定数）/ 明示型で宣言する。

```dart
var name = 'Dart';       // 型推論で String
final pi = 3.14;         // 再代入不可
const max = 100;         // コンパイル時定数
int count = 0;           // 明示型
```

関数は名前付き引数（`{}`）と位置オプション引数（`[]`）を持つ。名前付き引数は `required` を付けると必須になる。

```dart
String greet(String name, {String greeting = 'Hello', required int times}) {
  return List.filled(times, '$greeting, $name!').join(' ');
}

greet('A', times: 2);                  // 名前付き引数で呼ぶ
greet('B', greeting: 'Hi', times: 1);
```

カスケード演算子 `..` は同じオブジェクトに続けて操作する記法で、UI ツリーの構築で多用される。

```dart
final buffer = StringBuffer()
  ..write('a')
  ..write('b')
  ..writeln('c');   // 同じ buffer に連続適用
```

コレクション内で `if` / `for` / スプレッド `...` を直接使える。

```dart
final showExtra = true;
final items = [
  'base',
  if (showExtra) 'extra',
  for (var i = 0; i < 3; i++) 'item$i',
  ...['x', 'y'],
];
```

## この言語の核心概念（他言語と違う・必ず押さえる）
ここは Dart を Dart たらしめる中核。文法の細部より先に、この6点の「考え方」を押さえる。

### 健全な null safety（sound null safety）
①何か：型に「null を持てるか」を埋め込み、`?` なしの型には null を一切入れられないルール。`?.`（null なら飛ばす）・`??`（null のときの代替）・`!`（null でないと断言）・`late`（初期化を後回し）が道具。「健全（sound）」とは、コンパイラが null 非許容と判断した値は実行時にも絶対 null にならない、という保証のこと。

②具体コード：
```dart
String nonNull = 'ok';            // null 代入は不可
String? maybe;                    // 既定で null
int len = maybe?.length ?? 0;     // null なら 0
late String config;               // 後で必ず代入する約束
void setup() => config = 'ready'; // 初期化前に読むと実行時エラー
```

③他言語と違う点/つまずき：Java や旧来の言語では「どの参照も null になりうる」が既定で、null チェックは規律でしかない。Dart は型が保証する。`!` は「健全性を自分の責任で外す」操作なので、外した箇所が実際に null だと即クラッシュする。`late` も「初期化済みのはず」を人間が約束する仕組みで、破ると実行時に落ちる。

### すべてがオブジェクト（int も String も）
①何か：プリミティブ型が存在せず、`int`・`double`・`bool`・関数・`null`（`Null` 型の唯一の値）まで含めて全部がオブジェクト。全ての型は `Object?` を頂点とする一本の階層に属する。

②具体コード：
```dart
int n = 42;
print(n.isEven);           // int がメソッドを持つ
print(n.toString());       // どの値でもメソッド呼び出しできる
print(3.bitLength);        // リテラルに直接ドット
```

③他言語と違う点/つまずき：Java の `int`（プリミティブ）と `Integer`（オブジェクト）のような二重構造がない。ボクシング/アンボクシングを意識しなくてよい代わりに、数値も参照のように振る舞う。`Object?` が全ての頂点で、`Object`（非 null）と `Object?`（null 可）を取り違えると null safety の効き方が変わる。

### mixin（`with`）による振る舞いの合成
①何か：クラスは単一継承だが、`mixin` を `with` で複数取り込んで振る舞いを合成できる仕組み。状態とメソッドを持つ部品を、継承の縦の系列とは別に横から差し込む。

②具体コード：
```dart
mixin Logger {
  void log(String m) => print('[LOG] $m');
}
mixin Timestamped {
  DateTime get now => DateTime.now();
}
class Service with Logger, Timestamped {
  void run() => log('start at $now');   // 両方の振る舞いを得る
}
```

③他言語と違う点/つまずき：Java の単一継承＋インターフェース（実装は持てない）とも、多重継承（菱形問題）とも違う。`with A, B` は線形化され、同名メンバは後に書いた mixin が勝つ。`abstract class`（インスタンス化できない基底）と `mixin`（`with` 専用）は役割が別で、どちらを使うか最初に迷いやすい。

### `final` / `const` / `var` の違い
①何か：`var` は型推論する可変変数、`final` は一度だけ代入できる実行時の不変、`const` はコンパイル時に値が確定している定数。`final` と `const` はどちらも再代入不可だが、「いつ値が決まるか」が違う。

②具体コード：
```dart
var a = 1;                         // 推論で int、再代入可
final now = DateTime.now();        // 実行時に決まる不変
const max = 100;                   // コンパイル時定数
const list = [1, 2, 3];            // 中身ごと不変・正準化される
// const bad = DateTime.now();     // ← 実行時値なのでコンパイルエラー
```

③他言語と違う点/つまずき：`const` は「コンパイル時に計算しきれる」値専用なので、`DateTime.now()` のような実行時値には使えない（そこは `final`）。さらに `const` オブジェクトは正準化（同じ値は同一インスタンスに共有）される。`final` は参照の固定であって中身の固定ではなく、`final list = [1]` でも `list.add(2)` はできる。

### Future / Stream と async-await / await for
①何か：非同期の単一値が `Future<T>`、連続値が `Stream<T>`。`async`/`await` で Future を、`async*`/`yield` と `await for` で Stream を素直な逐次コードのように書く。

②具体コード：
```dart
Future<String> fetch() async {
  await Future.delayed(Duration(seconds: 1));
  return 'data';
}
Stream<int> counter() async* {
  for (var i = 0; i < 3; i++) yield i;   // 値を順に流す
}
Future<void> main() async {
  print(await fetch());
  await for (final n in counter()) print(n);  // 流れてくる値を逐次処理
}
```

③他言語と違う点/つまずき：Future は「単一の将来値」、Stream は「複数の将来値」で道具が分かれている点を最初に混同しやすい。`await` は単発の Future、`await for` は Stream を待つ。Dart のイベントループは単一スレッドで、`await` で待っている間も同じスレッドが他のイベントを処理する（重い同期計算は後述の isolate に逃がす）。

### isolate（メモリ非共有の並行）
①何か：Dart の並行単位。各 isolate は独立したメモリとイベントループを持ち、メモリを共有せず、メッセージのやり取りだけで通信する。重い計算を別 isolate に逃がす最短経路が `Isolate.run`。

②具体コード：
```dart
import 'dart:isolate';

Future<int> heavy() => Isolate.run(() {
  var sum = 0;
  for (var i = 0; i < 100000000; i++) sum += i;
  return sum;     // 別 isolate で計算、結果だけ戻る
});
```

③他言語と違う点/つまずき：Java や C++ のスレッドは同じメモリを共有し、ロックで競合を防ぐ。isolate は共有メモリ自体がないので、データ競合が原理的に起きない代わりに、大きなデータの受け渡しはコピーになる。渡すクロージャがキャプチャした値もコピー対象で、送れない型（一部のハンドル等）を含むと実行時に例外になる。

### records / patterns / sealed（3.x の三点セット）
①何か：`records` は複数値をまとめる軽量な匿名複合型、`patterns` は分解と条件分岐を兼ねる記法、`sealed` はサブタイプを同一ライブラリに閉じて `switch` の網羅をコンパイラに検査させる仕組み。3つが組み合わさって「データの形を型で表し網羅的に分岐する」スタイルになる。

②具体コード：
```dart
(int, String) lookup() => (42, 'found');   // record で複数値を返す

sealed class Shape {}
class Circle extends Shape { final double r; Circle(this.r); }
class Square extends Shape { final double s; Square(this.s); }

double area(Shape s) => switch (s) {        // pattern で分解
  Circle(:final r) => 3.14 * r * r,
  Square(:final s) => s * s,
  // sealed なので全ケースを書けば漏れをコンパイラが検査
};
```

③他言語と違う点/つまずき：record は構造的等価（フィールドが等しければ等しい）で、クラスのインスタンス等価とは挙動が違う。`sealed` でないクラスでは `switch` の網羅チェックが効かず、`default` がないと実行時に例外になりうる。これらは 3.0 で入った比較的新しい機能なので、古いコードや解説と混在しやすい。

## 型・データモデル
- 数値は `int` と `double`（どちらも `num` のサブタイプ）。文字列 `String`、真偽 `bool`、`List` / `Map` / `Set` が基本コレクション。
- すべてがオブジェクトで、`Object?` が頂点、`Never` が底の型。型推論が強く、明示型は境界や公開 API で付けると読みやすい。
- ジェネリクスを持つ。`List<int>`、`Map<String, int>` のように型引数を取る。

null safety は型システムに組み込まれている。`?` を付けない型は null を許容しない。

```dart
String nonNull = 'ok';       // null 代入不可
String? maybe = null;        // null 許容
int len = maybe?.length ?? 0; // ?. と ?? で安全に扱う
String forced = maybe!;      // ! は null でないと断言（外すと実行時例外）
```

クラスは単一継承だが、`mixin` で複数の振る舞いを合成できる。`with` で取り込む。

```dart
mixin Logger {
  void log(String msg) => print('[LOG] $msg');
}

class Service with Logger {
  void run() => log('running');
}
```

## この言語らしさ / 特徴的な機能
3.x で導入された3点が Dart らしさの中心になる。

records は複数値をまとめて返す軽量な匿名複合型。フィールドは位置でも名前でもアクセスできる。

```dart
(int, String) lookup() => (42, 'found');
final r = lookup();
print(r.$1);   // 42（位置フィールド）

({int code, String label}) named() => (code: 200, label: 'OK');
final n = named();
print(n.code); // 名前付きフィールド
```

patterns は分解（destructuring）と条件分岐を兼ねる。`switch` 式や `if-case` で使う。

```dart
final point = (3, 4);
final (x, y) = point;        // 分解代入

final result = switch (point) {
  (0, 0) => 'origin',
  (final a, 0) => 'on x axis at $a',
  (_, final b) => 'y is $b',
};
```

sealed class は同一ライブラリ内のサブタイプを閉じる。`switch` で全分岐を網羅していないとコンパイラが警告するため、状態の表現に向く。

```dart
sealed class Shape {}
class Circle extends Shape { final double r; Circle(this.r); }
class Square extends Shape { final double s; Square(this.s); }

double area(Shape shape) => switch (shape) {
  Circle(:final r) => 3.14 * r * r,
  Square(:final s) => s * s,
  // 全ケースを書くと default 不要、漏れがあれば警告
};
```

## 並行・非同期
非同期は `Future`（単一の将来値）と `Stream`（連続する値）を `async` / `await` で扱う。

```dart
Future<String> fetch() async {
  await Future.delayed(Duration(seconds: 1));
  return 'data';
}

Stream<int> counter() async* {
  for (var i = 0; i < 3; i++) {
    yield i;             // async* + yield でストリームを生成
  }
}

Future<void> main() async {
  print(await fetch());
  await for (final n in counter()) print(n);
}
```

並行は isolate を使う。isolate はメモリを共有せず、メッセージ受け渡しで通信するため、データ競合が原理的に起きない。重い処理を別 isolate に逃がすには `Isolate.run` が簡単。

```dart
import 'dart:isolate';

Future<int> heavy() => Isolate.run(() {
  var sum = 0;
  for (var i = 0; i < 100000000; i++) sum += i;
  return sum;     // 別 isolate で計算、結果だけ返る
});
```

スレッドとの違いは、共有メモリがない点。状態をロックで守る必要がなく、その代わり大きなデータの受け渡しはコピー（または一部は転送）になる。

## 標準ライブラリ / ツールチェーン
- `dart:core`（基本型）、`dart:async`（Future/Stream）、`dart:collection`、`dart:convert`（JSON）、`dart:io`（ファイル・プロセス、Web では使えない）、`dart:math`。
- パッケージ管理は `pub`。依存は `pubspec.yaml` に書き `dart pub get` で取得。公開レジストリは pub.dev。
- `dart format`（整形）、`dart analyze`（静的解析、`analysis_options.yaml` で lint 設定）、`dart test`（テスト）、`dart compile`（AOT 各種ターゲット）。

```yaml
# pubspec.yaml
dependencies:
  http: ^1.2.0
dev_dependencies:
  test: ^1.25.0
```

## このバージョンの新機能・トピック
- Dart 3.5：ネイティブ相互運用や Web 連携の改善が中心の安定リリース。言語構文の破壊的追加はなく、3.0 系で入った records / patterns / sealed の運用を安定させる方向。
- 3.3〜3.4 で extension types（ゼロコストの型ラッパ）が安定化しており、3.5 でも利用できる。既存の型に実行時コストなしで別の API を被せられる。

```dart
extension type UserId(int value) {
  bool get isValid => value > 0;   // int をラップした専用型
}
```

- 周辺の動きとして、将来の null 非許容を見据えた解析の厳格化や、`dart:js_interop`（型安全な JS 相互運用）への移行が進んでいる。

## ハマりどころ
- `!`（null 断言）の多用：null でないと断言した値が実際は null だと実行時にクラッシュする。`?.` と `??` で逃がすのが基本。
- `late` 変数：初期化を遅延させられるが、初期化前にアクセスすると実行時エラー。本当に遅延が必要な場合だけ使う。
- `const` と `final` の混同：`const` はコンパイル時に値が確定する必要がある。実行時に決まる値には `final` を使う。
- records の等価性：record は構造的等価（フィールドが等しければ等しい）。クラスのインスタンス等価とは挙動が違う。
- isolate へ渡すデータ：クロージャがキャプチャする値もコピー対象になり、送れない型（一部のハンドル等）を含むと例外になる。
- `switch` 式の網羅性：sealed でないクラスでは網羅チェックが効かず、`default` がないと実行時に例外になりうる。

## 関連
- [../flutter/](../flutter/) … Dart のフラッグシップであるクロスプラットフォーム UI フレームワーク Flutter の版別リファレンス。
- 同フォルダの [README.md](../README.md) … Dart 言語の概要・強み弱み・エコシステム。
