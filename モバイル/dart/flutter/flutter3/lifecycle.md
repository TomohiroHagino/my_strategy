# StatefulWidget のライフサイクル（initState / dispose）（Flutter 3）

## ひとことで言うと
`State` オブジェクトが「生まれてから消えるまで」に Flutter が順に呼ぶメソッド群。**`initState`（初期化）→ `build`（描画）→ … → `dispose`（後始末）** の流れを押さえると、リソース漏れや「mounted エラー」を防げる。

## 役割・なぜ必要か
- `StatefulWidget` は内部状態（`State`）を持つ。その State には「作られた時に1回だけやること」「再描画ごとにやること」「壊れる時に必ずやること」があり、それぞれ専用メソッドで受け取る。
- とくに **コントローラ・リスナ・タイマ・購読（Stream）は確保したら必ず解放**しないとメモリ漏れや多重コールバックになる。解放の置き場が `dispose`。
- 親から渡された値（`widget.xxx`）が変わった時、`InheritedWidget`（Theme など）が変わった時に再初期化したい、というフックも用意されている。

## 基本の書き方（コード）
```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key, required this.title});
  final String title;

  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  late final TextEditingController _controller;
  Timer? _timer;
  int _count = 0;

  @override
  void initState() {
    super.initState();                 // 最初に呼ぶ
    // 初期化：コントローラ生成、リスナ登録、購読開始など
    _controller = TextEditingController();
    _timer = Timer.periodic(const Duration(seconds: 1), (_) {
      if (mounted) setState(() => _count++); // async/遅延後は mounted を確認
    });
    // 注意：ここで context に依存する処理（Theme.of など）は避ける
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // InheritedWidget（Theme / MediaQuery / Localizations 等）に依存する初期化はここ
  }

  @override
  void didUpdateWidget(covariant CounterPage oldWidget) {
    super.didUpdateWidget(oldWidget);
    // 親から渡される widget が差し替わった時（title が変わった等）の追従処理
    if (oldWidget.title != widget.title) {
      // 必要なら再設定
    }
  }

  @override
  Widget build(BuildContext context) {
    // 状態が変わるたびに呼ばれる。重い処理や副作用はここに書かない
    return Scaffold(
      appBar: AppBar(title: Text(widget.title)),
      body: Center(child: Text('count: $_count')),
    );
  }

  @override
  void dispose() {
    // 後始末：initState で確保したものを必ず解放（順序は解放→super）
    _timer?.cancel();
    _controller.dispose();
    super.dispose();
  }
}
```

## 実務での使い方・定番パターン
- **`initState`**：State 生成時に1回だけ。コントローラ生成、リスナ登録、Stream購読開始、初回データ取得のキック。
- **`build`**：状態変化のたびに何度も呼ばれる前提で、**純粋に「現在の状態→UI」を返す**だけにする。
- **`didChangeDependencies`**：`initState` の直後にも呼ばれ、依存する `InheritedWidget` が変わるたびにも呼ばれる。`Theme.of(context)` 等に依存する初期化はここ。
- **`didUpdateWidget`**：同じ位置で `StatefulWidget` だけ差し替わった時に呼ばれる。`oldWidget` と現 `widget` を比べて差分追従（古い購読を解除し新しい値で張り直す等）。
- **`dispose`**：State 破棄時。`initState`/`didChangeDependencies` で確保したものを左右対称に解放。
- 非同期処理の結果で UI を触る前に **`if (!mounted) return;`** を入れるのが定石。

## ハマりどころ / アンチパターン
- **解放忘れ**：`dispose` で `controller.dispose()` / `removeListener` / `timer.cancel()` / `subscription.cancel()` をしないとメモリ漏れ・多重発火。**確保したものは必ず対で解放**。
- **`initState` で `context` 依存処理**（`Theme.of` / `MediaQuery.of` / `Provider.of` 等）：この時点では依存が確定しておらず例外や不正値になりうる。**`didChangeDependencies` へ移す**。`Navigator` 等を即時に使いたい時は `WidgetsBinding.instance.addPostFrameCallback` で次フレームへ。
- **async 後の `setState`**：`await` 中に画面が破棄されると `setState() called after dispose()` で落ちる。**`setState` 前に `mounted` を確認**。
- **`build` に副作用**：`build` 内で API 呼び出し・`setState`・購読開始をするとループや多重実行に。副作用は `initState` / イベントハンドラへ。
- **`super` の呼び忘れ／順序**：`initState`/`didChangeDependencies`/`didUpdateWidget` は先頭で `super` を呼ぶ。`dispose` は解放してから最後に `super.dispose()`。
- **`dispose` 後にコントローラ参照**：破棄済みオブジェクトへのアクセスは例外。タイマ等のコールバック内では `mounted` を見る。
- `late final` の初期化を `initState` でしないまま `build` で参照すると LateInitializationError。

## 関連
[state_management.md](./state_management.md) / [forms_input.md](./forms_input.md) / [widgets.md](./widgets.md)
