# 非同期（Future / async / FutureBuilder / Stream）（Flutter 3）

## ひとことで言うと
時間のかかる処理（通信・DB・ファイル）の結果を**待たずに先へ進み、終わったら受け取る**仕組み。1回きりの未来の値が **`Future`**、連続して流れてくる値が **`Stream`**。`async`/`await` で同期コードのように書ける。

## 役割・なぜ必要か
- Flutter の UI は単一スレッド（UI スレッド）で動く。通信を「待って止まる」と画面が固まる。だから非同期で逃がす。
- **`Future<T>`** = いつか1つの `T`（または失敗）を返す約束。`await` でその完了まで「論理的に待つ」（UI は止めない）。
- **`Stream<T>`** = 0個以上の値が時間差で流れてくる（位置情報の更新、WebSocket、入力イベントなど）。
- UI に載せるには非同期の結果を Widget に橋渡しする必要があり、それが **`FutureBuilder`**（Future用）と **`StreamBuilder`**（Stream用）。

## 基本の書き方（コード）
```dart
// 1) Future / async / await
Future<String> fetchUserName() async {
  await Future.delayed(const Duration(seconds: 1)); // 通信の代わり
  return 'Alice';
}

void example() async {
  try {
    final name = await fetchUserName(); // 完了まで待つ（UIは止まらない）
    print('hello $name');
  } catch (e) {
    print('失敗: $e'); // 例外は try/catch で受ける
  }
}
```

```dart
// 2) FutureBuilder：Future の結果を UI に出す（ローディング/エラー込み）
class UserName extends StatefulWidget {
  const UserName({super.key});
  @override
  State<UserName> createState() => _UserNameState();
}

class _UserNameState extends State<UserName> {
  late final Future<String> _future; // ← initState で“一度だけ”生成して保持

  @override
  void initState() {
    super.initState();
    _future = fetchUserName(); // build のたびに作らない（重要）
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<String>(
      future: _future,
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return const CircularProgressIndicator(); // ローディング
        }
        if (snapshot.hasError) {
          return Text('エラー: ${snapshot.error}'); // エラー
        }
        return Text('user: ${snapshot.data}'); // 成功
      },
    );
  }
}
```

```dart
// 3) StreamBuilder：流れてくる値を逐次表示
Stream<int> counter() async* {
  for (var i = 1; i <= 5; i++) {
    await Future.delayed(const Duration(seconds: 1));
    yield i; // 1秒ごとに値を流す
  }
}

Widget build(BuildContext context) {
  return StreamBuilder<int>(
    stream: counter(),
    builder: (context, snapshot) {
      if (!snapshot.hasData) return const CircularProgressIndicator();
      return Text('tick: ${snapshot.data}');
    },
  );
}
```

```dart
// 4) 非同期完了後の setState は mounted を確認
Future<void> _load() async {
  final name = await fetchUserName();
  if (!mounted) return;        // 画面が破棄済みなら触らない
  setState(() => _name = name);
}
```

## 実務での使い方・定番パターン
- **`async`/`await` を基本**にし、`Future.then(...)` のコールバック連結は避ける（読みやすさ重視）。
- **`FutureBuilder` の3状態を必ず描く**：ローディング / エラー / 成功。`snapshot.connectionState` と `hasError` / `hasData` で分岐するのが定番。
- **Future は `initState`（または provider/状態管理）で1回だけ生成**して `FutureBuilder` に渡す。これが安定動作の肝。
- **連続データは `StreamBuilder`**。位置情報・タイマー・Firestore・WebSocket などに。`StreamController` を使うときは `dispose` で `close()` する。→ [lifecycle.md](./lifecycle.md)
- **複数の Future を待つ**：並行実行は `await Future.wait([a, b])` でまとめて待つと速い。

## ハマりどころ / アンチパターン
- **`FutureBuilder` の `build` 内で毎回 `Future` を生成する**（最頻出バグ）：`future: fetchUserName()` のように `build` 直書きすると、再ビルドのたびに通信が走り、ローディングがチラつく／無限ループ的に再実行される。→ **`initState` で生成して変数に保持**。
- **`await` の後に `setState` / `context` を無条件で使う**：完了時には画面が dispose 済みかもしれない。`if (!mounted) return;`（`context.mounted` も）で必ずガードする。→ [state_management.md](./state_management.md)
- **エラー/ローディングの未処理**：成功パスしか書かず、失敗時に空白や例外落ち。`hasError` を必ず分岐し、ユーザー向けメッセージを出す。
- **例外の握り潰し**：`try { ... } catch (_) {}` で握ると原因不明の不具合に。ログ＋UI 表示を最低限残す。
- **Stream の閉じ忘れ**：自前 `StreamController` や subscription を `dispose` で `close()`/`cancel()` しないとリーク。
- **重い同期処理を await しても軽くならない**：`await` は I/O 待ちを逃がすだけ。CPU 重処理は `compute`（Isolate）へ。

## 関連
[networking.md](./networking.md) / [state_management.md](./state_management.md) / [lifecycle.md](./lifecycle.md)
