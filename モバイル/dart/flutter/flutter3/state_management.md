# 状態管理（State Management）（Flutter 3）

## ひとことで言うと
UI を組み立てる元になる「状態（State）」を、**誰がどこに持ち、変わったときにどの範囲を再ビルドするか**を決める設計のこと。Flutter は「状態が変わる → `build()` が呼ばれて画面が再構築される」が基本なので、状態の置き場所がそのままアプリ構造になる。

## 役割・なぜ必要か
- Flutter の UI は宣言的（= 状態から画面を毎回計算する）。だから「**状態をどう更新するか**」が中心課題になる。
- 小さい画面なら `setState` で十分だが、画面をまたいで共有する状態（ログインユーザー、カート、設定）が出てくると、Widget ツリー越しに値を配り直す手段が要る。
- 規模に応じて段階的に選ぶ：**`setState`（ローカル最小）→ `InheritedWidget`（配布の仕組み）→ Provider / Riverpod / Bloc（本格的な共有・分離）**。

## 基本の書き方（コード）
```dart
// 1) setState：StatefulWidget の「自分の中だけ」の状態を最小で扱う
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _count = 0; // ← この Widget が持つローカル状態

  void _increment() {
    setState(() {       // setState の中で状態を変える
      _count += 1;      // → この State を持つ範囲だけ build() が再実行される
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(child: Text('Count: $_count')),
      floatingActionButton: FloatingActionButton(
        onPressed: _increment,
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

```dart
// 2) 状態を上位に持ち上げる（lifting state up）
// 子が状態を持たず、親が持って「値」と「変更関数」を渡す
class ParentPage extends StatefulWidget {
  const ParentPage({super.key});
  @override
  State<ParentPage> createState() => _ParentPageState();
}

class _ParentPageState extends State<ParentPage> {
  int _count = 0; // 状態は親に集約

  @override
  Widget build(BuildContext context) {
    return Column(children: [
      Text('$_count'),
      // 子は表示と通知だけ。状態は持たない（presentational）
      AddButton(onPressed: () => setState(() => _count++)),
    ]);
  }
}

class AddButton extends StatelessWidget {
  const AddButton({super.key, required this.onPressed});
  final VoidCallback onPressed;
  @override
  Widget build(BuildContext context) =>
      ElevatedButton(onPressed: onPressed, child: const Text('+1'));
}
```

```dart
// 3) Riverpod（推奨されやすい）：状態を Widget ツリーの外に置いて共有
// flutter pub add flutter_riverpod
final counterProvider = StateProvider<int>((ref) => 0);

class RiverpodCounter extends ConsumerWidget {
  const RiverpodCounter({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider); // 変化を購読 → ここだけ再ビルド
    return Column(children: [
      Text('$count'),
      ElevatedButton(
        onPressed: () => ref.read(counterProvider.notifier).state++,
        child: const Text('+1'),
      ),
    ]);
  }
}
// アプリ全体を ProviderScope で囲む必要がある（main で runApp(ProviderScope(child: App()))）
```

## 実務での使い方・定番パターン
- **まず `setState`**：その画面だけで完結する状態（トグル、入力中の値、開閉）は迷わずローカルに。外部ライブラリ不要。
- **共有が出たら持ち上げる（lifting）**：兄弟Widget間で値を共有したいなら、共通の親へ状態を上げて props で配る。これで足りるケースは多い。
- **配布が辛くなったら Provider / Riverpod**：親から孫へバケツリレー（prop drilling）が深くなったら導入の合図。`InheritedWidget` はその「ツリー越し配布」の素の仕組みで、Provider 系はそれを使いやすく包んだもの。
- **選び方の目安**：手軽さ重視なら **Provider**、型安全・テスト容易・`BuildContext` 非依存を重視するなら **Riverpod**、イベント/状態を厳格に分離したい大規模なら **Bloc**。新規は Riverpod が無難。
- **再ビルド範囲を絞る**：`Consumer` / `ref.watch` を「本当に使う最小の Widget」に置く。上位で watch すると広範囲が再ビルドされて重くなる。

## ハマりどころ / アンチパターン
- **`setState` の乱用 + 巨大 Widget**：1つの `build` に何百行も詰め、`setState` で全体を再描画 → 重い・読めない。Widget を分割し、状態は必要な最小単位へ。→ [widgets.md](./widgets.md)
- **状態の置き場所ミス**：本来ローカルでよい状態をグローバルに置く（逆も）。「**どこから見えれば良いか**」で置き場所を決める。共有不要ならローカルが正解。
- **`setState` を `build` 内や非同期完了後に無条件で呼ぶ**：`build` 中の `setState` は例外。非同期完了後は破棄済みの可能性があるので `mounted` を確認する。→ [async_futures.md](./async_futures.md) / [lifecycle.md](./lifecycle.md)
- **ライブラリ選定で迷い続ける**：小さいうちは `setState`＋lifting で進め、痛くなってから Riverpod を入れる。最初から全部 Bloc 化するのは過剰になりやすい（YAGNI）。
- **再ビルド範囲を意識しない**：watch/Consumer を上位に置きすぎてカクつく。表示に使う直近の Widget で購読する。

## 関連
[widgets.md](./widgets.md) / [lifecycle.md](./lifecycle.md) / [navigation.md](./navigation.md) / [async_futures.md](./async_futures.md)
