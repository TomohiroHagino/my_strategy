# フォーム / 入力（TextField / Form）（Flutter 3）

## ひとことで言うと
ユーザーからの文字入力を受け取る仕組み。単発の入力欄が **`TextField`**、複数欄をまとめて**まとめてバリデーション**できるのが **`Form` + `TextFormField`**。入力値の取得・監視には **`TextEditingController`** か `onChanged` を使う。

## 役割・なぜ必要か
- ログイン・検索・投稿フォームなど、**ユーザー入力はアプリの主要な入口**。ここで取った値をサーバー送信や状態更新につなぐ。
- 入力は「**信頼できない外部データ**」。送信前に**必須・形式・長さ**を検証する場所が要る。`Form` + `validator` がその仕組み。
- 入力中の値を**プログラム側で読み書き**（初期値設定・クリア・他欄連動）するために `TextEditingController` が必要になる。

## 基本の書き方（コード）
```dart
// 1) TextField + controller（単発の入力欄）
class SearchBox extends StatefulWidget {
  const SearchBox({super.key});
  @override
  State<SearchBox> createState() => _SearchBoxState();
}

class _SearchBoxState extends State<SearchBox> {
  // controller は State のフィールドとして1回だけ生成する
  final _controller = TextEditingController();

  @override
  void dispose() {
    _controller.dispose(); // ★必須：忘れるとメモリリーク
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: _controller,
      decoration: const InputDecoration(labelText: '検索ワード'),
      onChanged: (text) {
        // 1文字ごとに呼ばれる（リアルタイム反映に使う）
        debugPrint('入力中: $text');
      },
      onSubmitted: (text) => debugPrint('確定: $text'),
    );
  }
}
```

```dart
// 2) Form + TextFormField + validator + GlobalKey
class LoginForm extends StatefulWidget {
  const LoginForm({super.key});
  @override
  State<LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<LoginForm> {
  // Form の状態を外から操作するための鍵
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();

  @override
  void dispose() {
    _emailController.dispose();
    super.dispose();
  }

  void _submit() {
    // 全 TextFormField の validator をまとめて実行
    if (_formKey.currentState!.validate()) {
      debugPrint('OK: ${_emailController.text}');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            controller: _emailController,
            decoration: const InputDecoration(labelText: 'メール'),
            keyboardType: TextInputType.emailAddress,
            // null を返せばOK、文字列を返すとそれがエラー表示
            validator: (value) {
              if (value == null || value.isEmpty) return '必須です';
              if (!value.contains('@')) return '形式が正しくありません';
              return null;
            },
          ),
          ElevatedButton(onPressed: _submit, child: const Text('送信')),
        ],
      ),
    );
  }
}
```

## 実務での使い方・定番パターン
- **値の取り方は2系統**：①`controller.text` で取る（送信時にまとめて読む向き）②`onChanged` で都度受け取り `setState` や状態管理に流す（リアルタイム表示・絞り込み向き）。両方を同じ値に混在させない。
- **`Form` でまとめて検証**：複数欄は `Form` で囲み、`_formKey.currentState!.validate()` で一括チェック。`autovalidateMode: AutovalidateMode.onUserInteraction` を付けると入力しながらエラー表示できる。
- **送信後にクリア**：`_controller.clear()`。初期値を入れたいなら `TextEditingController(text: '初期値')`。
- **パスワード欄**：`obscureText: true`。表示切替は `setState` で `obscureText` をトグル。
- **整形・桁数制限**：`inputFormatters`（数字のみ等）、`maxLength` で文字数制限。
- **キーボード制御**：`keyboardType`（数値・メール）、`textInputAction`（次へ/完了）、`FocusNode` で次の欄へフォーカス移動。
- **バリデーションの本体ロジック**は再利用しやすいよう関数に切り出し、`validator` から呼ぶ。

## ハマりどころ / アンチパターン
- **`TextEditingController` の `dispose` 忘れ＝メモリリーク**。`StatefulWidget` の `State` で生成したら、必ず `dispose()` で破棄する。画面を開閉するたびに増えていく。→ [lifecycle.md](./lifecycle.md)
- **`build` の中で `TextEditingController()` を生成しない**。再ビルドのたびに作り直され入力が消える／リークする。必ず `State` のフィールドで1回だけ生成する。
- **`Form` の `key` 付け忘れ／別の `GlobalKey` を使い回し**で `validate()` が効かない・例外。`GlobalKey<FormState>` は1つの `Form` に1つ。
- **`validate()` を呼ばずに送信**：`validator` を書いても呼ばなければ検証されない。送信ハンドラの先頭で必ず `validate()`。
- **同じ `GlobalKey` を複数 `Form` に使う**と「同一キーが複数ツリーにある」エラー。
- **`TextField`（`Form` 非対応）に `validator` を期待**：`validator` を持つのは `TextFormField`。`Form` 連携が要るなら後者を使う。
- **controller と `onChanged` で二重に状態管理**して値がズレる。どちらか一方を真実の源にする。
- 数値入力で `controller.text`（String）を**そのまま計算に使う** → `int.tryParse` で変換し、失敗時を必ず handle する。

## 関連
[lifecycle.md](./lifecycle.md) / [state_management.md](./state_management.md) / [styling_theming.md](./styling_theming.md)
