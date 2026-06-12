# データの流れ・各部分は何を返すか（Flutter 3）

## ひとことで言うと
モバイルは「サーバのリクエスト層」ではなく、**状態→UIの一方向データフロー**で考える。**ユーザー操作 → State が変わる → build() が再実行される → Widget ツリーが作り直される → 描画**。データ取得（API）は「状態を更新する手段」であり、最終的にすべて State 経由で UI に反映される。

## 全体の流れ（図）
```
ユーザー操作（タップ／入力）
   │
   ▼
[State 更新]   setState(...) / Notifier.state = ... / Riverpod の state 変更
   │           ※ ここで「画面を直接いじる」のではなく「状態を変える」だけ
   ▼
[build() 再実行]  状態を読んで Widget を組み立て直す（純粋関数的）
   │
   ▼
[Widget ツリー]   新ツリーを返す → Flutter が差分描画
   │
   ▼
  画面（描画）

── データ取得フロー（横入り）──────────────
[API / Repository]  Future/Stream で取得（http, dio）
   │ データ（モデル）を返す
   ▼
[State 更新]  取得結果を状態に格納（FutureBuilder / setState / Riverpod）
   │
   ▼  以降は上と同じ：build() 再実行 → 再描画
```
一方向（状態 → UI）。UI から状態へ戻る矢印は無く、操作は「次の状態」を作るだけ。

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 役割 |
|---|---|---|---|
| **ユーザー操作** | タップ／入力イベント | （ハンドラ呼び出し） | 状態変更のきっかけ |
| **API / Repository** | id / 条件 | **データ（モデル）**（Future/Stream） | 外部からデータ取得 |
| **State** | 操作・取得結果 | **build() へ渡る現在値** | 画面の元になる唯一の真実 |
| **build()** | State（と props） | **Widget ツリー（UI）** | 状態から画面を組み立てる |
| **Widget ツリー** | — | 描画される UI そのもの | 運ばれる側（毎回 new） |

- **State が「真実」**：画面は state の写像。state を変えれば UI が追従する。
- **build() は状態の関数**：副作用を書かず「今の状態ならこの UI」を返すだけ。

## コードで通して見る
```dart
// パターンA：StatefulWidget ＋ setState
class CounterPage extends StatefulWidget {
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _count = 0;                 // State（唯一の真実）

  void _increment() {
    setState(() => _count++);     // 操作 → State 更新（build が再実行される）
  }

  @override
  Widget build(BuildContext context) {        // State を受け取り…
    return Scaffold(
      body: Center(child: Text('$_count')),    // …Widget ツリー（UI）を返す
      floatingActionButton: FloatingActionButton(
        onPressed: _increment,                 // ユーザー操作 → _increment
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

```dart
// パターンB：Riverpod（Provider → ref.watch）＋ データ取得
final userProvider = FutureProvider<User>((ref) async {
  return ref.read(repositoryProvider).fetchUser();  // API/Repository → データを返す
});

class UserView extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(userProvider);   // State（取得結果）を watch
    return state.when(                        // 状態に応じて Widget を返す
      data: (user) => Text(user.name),        // 取得成功 → UI
      loading: () => const CircularProgressIndicator(),
      error: (e, _) => Text('エラー: $e'),
    );
  }
}
```

## 実務での使い方・定番パターン
- **状態を一箇所に集める**：散らばった `setState` より、ViewModel/Notifier（Riverpod 等）に寄せると追いやすい。→ [state_management.md](./state_management.md)
- **取得は Future/Stream → 状態 → build**：`FutureBuilder`／`StreamBuilder`／Riverpod の `AsyncValue` で「読み込み中・成功・失敗」を状態として扱う。→ [async_futures.md](./async_futures.md) / [networking.md](./networking.md)
- **UI を直接いじらない**：常に「状態を変える」→ Flutter が再描画。命令的な DOM 操作の発想を持ち込まない。

## ハマりどころ / アンチパターン
- **build() の中で副作用**：build で API 呼び出し・setState すると無限ループ。取得は initState / Future / `.task` 相当の場所で。
- **setState の付け忘れ**：変数だけ変えても再描画されない。状態変更は必ず setState / Notifier 経由。
- **巨大な build()**：ツリーが肥大化すると再ビルドが重い。Widget を分割し、変わる部分だけ再構築させる。
- **State をグローバル変数で代用**：再描画のトリガにならない。フレームワークの状態機構に乗せる。

## 関連
[state_management.md](./state_management.md) / [lifecycle.md](./lifecycle.md) / [async_futures.md](./async_futures.md) / [networking.md](./networking.md) / [widgets.md](./widgets.md)
