# ナビゲーション（Navigation）（Flutter 3）

## ひとことで言うと
画面（ページ）を**スタック（積み重ね）**として管理し、新しい画面を上に積む（push）／戻る（pop）仕組み。標準は `Navigator`、宣言的に書ける推奨ライブラリが **`go_router`**。

## 役割・なぜ必要か
- モバイルアプリは「一覧 → 詳細 → 編集 …」と画面を行き来する。この遷移と「戻る」を管理するのがナビゲーション。
- `Navigator` は画面を **後入れ先出し（LIFO）のスタック**として持つ。`push` で1枚積み、`pop` で1枚外す（= 戻る）。Android の戻るボタンもこの pop に繋がる。
- 画面が増えると `push`/`pop` の手書きや文字列ルートが破綻しがち。**`go_router`** は URL ライクなパスで遷移を一元管理でき、ディープリンクや Web URL とも相性が良い。

## 基本の書き方（コード）
```dart
// 1) Navigator.push / pop：最も素朴な命令的遷移
// 詳細画面へ進む
ElevatedButton(
  onPressed: () {
    Navigator.push(
      context, // ← BuildContext が必須（Navigator は context から辿る）
      MaterialPageRoute(builder: (context) => const DetailPage()),
    );
  },
  child: const Text('詳細へ'),
);

// 詳細画面側：戻る
ElevatedButton(
  onPressed: () => Navigator.pop(context), // 1枚外す＝前の画面へ
  child: const Text('戻る'),
);
```

```dart
// 2) 画面間の引数渡し（コンストラクタで渡すのが基本・型安全）
Navigator.push(
  context,
  MaterialPageRoute(builder: (_) => DetailPage(userId: 42)),
);

class DetailPage extends StatelessWidget {
  const DetailPage({super.key, required this.userId});
  final int userId;
  @override
  Widget build(BuildContext context) =>
      Scaffold(body: Center(child: Text('user: $userId')));
}

// 結果を受け取る（pop で値を返す）
final result = await Navigator.push<String>(
  context,
  MaterialPageRoute(builder: (_) => const PickerPage()),
);
// PickerPage 側： Navigator.pop(context, 'selected');
```

```dart
// 3) 名前付きルート（小〜中規模で使える標準機能）
MaterialApp(
  routes: {
    '/': (context) => const HomePage(),
    '/detail': (context) => const DetailPage(userId: 0),
  },
);
Navigator.pushNamed(context, '/detail'); // 文字列で遷移
```

```dart
// 4) go_router（推奨・宣言的）：パスで遷移を一元定義
// flutter pub add go_router
final router = GoRouter(
  routes: [
    GoRoute(path: '/', builder: (context, state) => const HomePage()),
    GoRoute(
      path: '/user/:id', // パスパラメータで引数を渡せる
      builder: (context, state) =>
          DetailPage(userId: int.parse(state.pathParameters['id']!)),
    ),
  ],
);
// main:  MaterialApp.router(routerConfig: router)
// 遷移:  context.go('/user/42');   // スタックを置き換え
//        context.push('/user/42'); // 上に積む（戻れる）
```

## 実務での使い方・定番パターン
- **小規模は `Navigator.push` + コンストラクタ引数**で十分。引数は名前付きルートの文字列より**コンストラクタ渡しが型安全**で安全。
- **中〜大規模・Web・ディープリンクありなら `go_router`**。パスで全画面を宣言的に定義でき、ネスト（ボトムナビ内の遷移）や認証ガード（リダイレクト）も書きやすい。
- **戻り値が欲しい遷移**は `Navigator.push<T>` を `await` し、相手側で `pop(context, value)`。検索結果や選択値の受け渡しに定番。
- **スタックを丸ごと入れ替える**（ログイン後にログイン画面へ戻らせない等）：`pushReplacement` / `pushAndRemoveUntil`、go_router なら `context.go(...)`。
- **モーダルやボトムシート**は `showDialog` / `showModalBottomSheet`。これも内部的に Navigator を使う。

## ハマりどころ / アンチパターン
- **push と pop の対応が崩れる**：push した回数だけ pop されないとスタックが詰まる/空 pop でクラッシュ。条件分岐の各経路で対応を揃える。空になり得るなら `Navigator.canPop(context)` で確認。
- **`BuildContext` 取り違え**：`Navigator.of(context)` は「その context が属するツリーの Navigator」を辿る。`MaterialApp` より上の context や、すでに dispose された context を使うとエラー。`Builder` で正しい context を作る。
- **非同期の後に context を使う**：`await` を挟むと画面が消えている可能性。`if (!context.mounted) return;` を入れてから遷移する。→ [async_futures.md](./async_futures.md)
- **名前付きルートに動的引数を素直に渡せない**：`pushNamed` の `arguments` は `Object?` で型が緩い。型安全にしたいなら go_router か直接コンストラクタ渡しへ。
- **戻る挙動の想定漏れ**：Android 物理戻る・iOS スワイプ戻る・`WillPopScope`/`PopScope` の制御を忘れると、確認ダイアログなしで離脱されるなど UX が崩れる。
- **go_router 導入後も `Navigator.push` を混在**：遷移経路が二系統になり戻る挙動が壊れやすい。導入したら原則 go_router の API に寄せる。

## 関連
[state_management.md](./state_management.md) / [async_futures.md](./async_futures.md) / [widgets.md](./widgets.md)
