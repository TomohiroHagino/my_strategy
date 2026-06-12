# mocktail（Flutter 3）

## ひとことで言うと
Dart 用のモック生成ライブラリ。テスト対象が依存するクラス（リポジトリ・APIクライアント等）の**偽物（モック）**を作り、戻り値を `when(() => …).thenReturn(…)` で固定し、呼ばれ方を `verify(() => …)` で検証する。`mockito` と違いコード生成（build_runner）が不要で、`Mock` を継承するだけで使える。

## 役割・なぜ必要か
- widget/unit テストで**本物のネットワークやDBを叩かない**ために、依存をモックに差し替える。
- 「成功」「失敗」「空」などの戻り値を**テスト側から自由に作れる**ので、エラー画面やローディングの分岐を確実に通せる。
- 依存先のメソッドが**期待通り（正しい引数で・正しい回数）呼ばれたか**を `verify` で保証できる。
- `mockito` のようなアノテーション＋`build_runner` のコード生成が不要で、手書きで完結する（コード生成の待ち時間・生成ファイルの管理が消える）。

## 基本の書き方（コード）
```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.0
```
```dart
// 対象コード（例）
abstract class UserRepository {
  Future<String> fetchName(int id);
}

class Greeter {
  Greeter(this.repo);
  final UserRepository repo;
  Future<String> greet(int id) async => 'Hello, ${await repo.fetchName(id)}';
}
```
```dart
// test/greeter_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

// 1) Mock を継承するだけ（コード生成なし）
class MockUserRepository extends Mock implements UserRepository {}

void main() {
  test('greet はリポジトリの名前で挨拶を返す', () async {
    // Arrange
    final repo = MockUserRepository();
    // 2) 戻り値を固定。引数は any() でも具体値でもよい
    when(() => repo.fetchName(1)).thenAnswer((_) async => 'Taro'); // Future は thenAnswer
    final greeter = Greeter(repo);

    // Act
    final result = await greeter.greet(1);

    // Assert
    expect(result, 'Hello, Taro');
    // 3) 呼ばれ方を検証（正しい引数で1回）
    verify(() => repo.fetchName(1)).called(1);
  });

  test('失敗を再現する', () async {
    final repo = MockUserRepository();
    when(() => repo.fetchName(any())).thenThrow(Exception('network')); // 例外を投げさせる
    final greeter = Greeter(repo);
    expect(() => greeter.greet(1), throwsException);
  });
}
```
```dart
// 同期の戻り値は thenReturn、Future は thenAnswer
when(() => calc.add(1, 2)).thenReturn(3);                 // 同期
when(() => api.load()).thenAnswer((_) async => <int>[]); // 非同期
```

## 実務での使い方・定番パターン
- **`thenReturn` と `thenAnswer` の使い分け**：同期の値は `thenReturn`、`Future`/`Stream` は `thenAnswer((_) async => …)`。Future に `thenReturn` を使うと型は通っても遅延評価されず崩れやすい。
- **引数マッチャ `any()`**：引数を問わず固定したいときに。位置引数は `any()`、名前付き引数は `any(named: 'id')`。
- **`registerFallbackValue`**：`any()` でカスタム型（例：`User`）を使うときは、`setUpAll` で `registerFallbackValue(FakeUser())` を**事前登録**しないと実行時エラー。プリミティブ型（int/String等）は不要。
- **`verify` / `verifyNever` / `verifyInOrder`**：呼ばれた回数・順序を検証。`captureAny()` で実際に渡された引数を取り出してアサートもできる。
- **state 管理と組み合わせ**：Provider/Riverpod のリポジトリをモックに差し替えて注入し、widget test で UI 分岐（成功/エラー/空）を網羅する。→ [testing.md](./testing.md)
- **`reset(mock)`**：テスト間でモック状態を初期化（`setUp` で新規生成する方が安全で定番）。

## ハマりどころ
- **`registerFallbackValue` 忘れ**：`any()` に独自クラスを渡すと `Bad state: ... fallback value` で落ちる。`setUpAll` で `registerFallbackValue(...)` を登録する。
- **`when` を関数で囲み忘れ**：mocktail は `when(() => repo.x())` のように**クロージャ**で渡す（mockito の `when(repo.x())` とは構文が違う）。素で呼ぶと値が評価されてしまう。
- **Future に `thenReturn`**：`thenReturn(Future.value(...))` でも動く場面はあるが、例外を投げさせたいときなどで崩れる。`thenAnswer` に統一が安全。
- **`implements` 対象に存在しないメソッド**：抽象クラス/インターフェース通りに `implements` しないと型エラー。具象クラスを直接 mock すると final メソッド等で問題が出やすいので、依存は**抽象に対して**切る。
- **検証し過ぎ**：内部の呼び出し回数まで縛ると、リファクタで赤くなる「もろい」テストに。境界（API/DB）だけ検証する。
- **`verify` の二重消費**：`verify` は呼び出し記録を消費する。同じ呼び出しを2回 `verify` すると2回目が落ちる。

## 関連
[testing.md](./testing.md) / [state_management.md](./state_management.md) / [networking.md](./networking.md)
