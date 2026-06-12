# Widget（Stateless / Stateful）（Flutter 3）

## ひとことで言うと
画面を構成する**部品の最小単位**。Flutter では文字・余白・ボタン・画面そのものまで「**すべてが Widget**」で、Widget を入れ子にした **Widget ツリー**が UI になる。

## 役割・なぜ必要か
- Flutter の UI は「Widget の組み合わせ」で宣言的に書く。`build()` が「**今の状態ならこう描く**」という設計図を返し、Flutter がそれを実際のピクセルに変換する。
- Widget には2種類あり、**状態を持つか**で使い分ける。
  - **StatelessWidget**：状態なし。渡された値（引数）だけで見た目が決まる。`build()` のみ。
  - **StatefulWidget**：状態あり。内部で変化する値を持ち、`setState()` で「状態が変わった→ build し直して」と Flutter に伝える。
- 「状態が変われば再ビルドされる」のがFlutterのUIモデル。だから**何を状態として持つか**の設計が UI 設計の中心になる。

## 基本の書き方（コード）
```dart
import 'package:flutter/material.dart';

// 状態なし：引数(label)だけで見た目が決まる。build() だけ
class Badge extends StatelessWidget {
  final String label;
  const Badge({super.key, required this.label});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      color: Colors.blue.shade100,
      child: Text(label),
    );
  }
}

// 状態あり：内部の count が変わる → setState → build し直し
class Counter extends StatefulWidget {
  const Counter({super.key});

  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0; // ← これが「状態(State)」。再ビルドされても保持される

  void _increment() {
    setState(() {       // setState の中で状態を書き換える
      _count++;         // これが build() の結果に反映される
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Text('count: $_count'),
        ElevatedButton(onPressed: _increment, child: const Text('+1')),
      ],
    );
  }
}
```

## 実務での使い方・定番パターン
- **まず Stateless で書く**。「この Widget の中で時間とともに変わる値があるか？」を問い、**ある時だけ Stateful** にする。迷ったら Stateless（軽い・テストしやすい）。
- **状態は持ち主を1か所に**。複数 Widget で共有する状態は、Stateful の中に閉じ込めず、状態管理（Provider / Riverpod 等）へ引き上げる。→ [state_management.md](./state_management.md)
- **`const` を付ける**：引数が固定の Widget は `const Badge(label: 'NEW')` のように `const` 化すると、親が再ビルドされても**そのサブツリーは作り直されない**（性能に効く）。
- **小さく分割**：1つの巨大な `build()` ではなく、意味のある単位で Widget に切り出すと、再ビルド範囲が狭まり読みやすい。
- **`BuildContext` は「ツリー上の現在位置」**。`Theme.of(context)` などで親の情報を取りに行く入口になる。→ [styling_theming.md](./styling_theming.md)

## ハマりどころ / アンチパターン
- **`setState` を忘れて値だけ変える** → 画面が更新されない。状態の更新は必ず `setState(() {...})` 内で。
- **`build()` 内で重い処理をする**：`build()` は何度も呼ばれる前提。API 通信・重い計算・コントローラ生成をここに書くと**毎ビルド走って**重い/バグる。生成は `initState`、副作用は別所へ。→ [lifecycle.md](./lifecycle.md)
- **何でも Stateful**：状態が要らないのに Stateful にすると無駄に複雑。Stateless で済むなら Stateless。
- **`const` 付け忘れで再ビルドが増える**：固定の Widget に `const` を付けないと、親の再ビルドのたびに作り直される。
- **`build` の戻りで分岐を散らかす**：深いネストになりがち。小 Widget に切り出して早期 return で平らにする。→ [layout.md](./layout.md)

## 関連
[layout.md](./layout.md) / [state_management.md](./state_management.md) / [lifecycle.md](./lifecycle.md)
