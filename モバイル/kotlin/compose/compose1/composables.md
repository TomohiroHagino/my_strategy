# Composable 関数（Jetpack Compose）

## ひとことで言うと
`@Composable` を付けた関数で、**UIの一部を「関数」として宣言する**もの。引数（状態）を受け取り、それに対応する画面を描く。状態が変わると**再コンポーズ（recomposition）**で描き直される。

## 役割・なぜ必要か
- Compose の世界では「ボタン」「リスト」「画面」がすべて `@Composable fun` で表現される。**UI＝状態を引数に取り、画面を返す関数**という考え方が核。
- 命令的に「View を取得して setText する」のではなく、**状態を渡せば対応する見た目になる**（宣言的UI）。状態が変われば Compose が差分を描き直す＝**再コンポーズ**。
- 関数なので**組み合わせ・再利用・テスト**がしやすい。小さな部品を合成して画面を作る。
- 「いま何を表示すべきか」を関数の入力（引数）で決めるので、**見た目と状態がズレない**。

## 基本の書き方（コード）
```kotlin
// 命名は PascalCase（型のように扱う）。返り値なし＝Unit
@Composable
fun UserCard(name: String, online: Boolean) {
    // 引数だけで見た目が決まる＝stateless（状態を持たない）
    Column {
        Text(name)
        Text(if (online) "オンライン" else "オフライン")
    }
}

// 部品を合成して大きなUIを作る
@Composable
fun UserList(users: List<User>) {
    Column {
        for (u in users) {
            UserCard(name = u.name, online = u.online)
        }
    }
}
```

```kotlin
// 状態を「持つ」例（stateful）。後で state.md で深掘り
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }   // 状態
    Button(onClick = { count++ }) {               // 状態を変える
        Text("count = $count")                    // 状態に追従して再コンポーズ
    }
}
```

## 実務での使い方・定番パターン
- **命名は PascalCase**（`UserCard`）。`@Composable` 関数は「UI部品の型」のように扱う慣習。`Unit` を返し、値ではなく画面を“出す”。
- **stateless を基本にする**：状態は引数で受け取り、見た目だけ描く部品にしておくと再利用・プレビュー・テストが楽。状態を持たせるのは上位の少数に絞る。→ [state.md](./state.md)
- 状態を持つ部品（stateful）と描くだけの部品（stateless）を分け、**状態は上へ持ち上げる（状態ホイスティング）**のが定番。→ [state.md](./state.md)
- 並べる・重ねるは `Column`/`Row`/`Box` で行い、見た目の調整は `Modifier` を引数で渡す。→ [layout.md](./layout.md)
- 大きくなったら**意味のある単位で関数に分割**。1関数1責務にすると `@Preview` も付けやすい。

## ハマりどころ / アンチパターン
- **`@Composable` 内で副作用を直書きしない**。「ネットワーク呼び出し」「ログ送信」「`var` を直接書き換え」などを本体に書くと、再コンポーズのたびに何度も走る。副作用は **`LaunchedEffect` / `rememberCoroutineScope` 等**に逃がす。→ [async_coroutines.md](./async_coroutines.md)
- **再コンポーズの回数・順序を前提にしない**。再コンポーズは**何度でも・順不同で・必要なければスキップ**され得る。「1回だけ実行される」前提のコードはバグる。
- **`@Composable` 本体で重い計算を毎回やらない**。結果は `remember` でキャッシュする。
- **ローカル `var` に状態を入れても保持されない**。再コンポーズで初期値に戻る。状態は `remember { mutableStateOf(...) }` で持つ。→ [state.md](./state.md)
- 関数名を camelCase（`userCard`）にする／`@Composable` の付け忘れ、は定番ミス。プレビューや呼び出しで気づく。

## 関連
[state.md](./state.md) / [layout.md](./layout.md)
