# リスト（LazyColumn / LazyRow）（Jetpack Compose）

## ひとことで言うと
**画面に見えている分だけを描画する遅延リスト**。縦は `LazyColumn`、横は `LazyRow`。RecyclerView の Compose 版にあたり、要素は `items(...) { }` ブロックの中で組み立てる。

## 役割・なぜ必要か
- 通常の `Column` / `Row` は **渡した全要素を一度に measure・layout・描画**する。10件なら問題ないが、1000件・無限スクロールでは全部を作ってしまい、メモリと初回描画が重くなる。
- `LazyColumn` は **可視範囲（＋少しの先読み）だけをコンポーズ**し、スクロールで画面外に出た要素は破棄・再利用する。これが「Lazy（遅延）」の意味。
- つまり「**件数が多い／可変／スクロールする**」なら Lazy、「**数件で固定**」なら普通の `Column` で十分、という住み分け。

## 基本の書き方（コード）
```kotlin
@Composable
fun UserList(users: List<User>, modifier: Modifier = Modifier) {
    LazyColumn(
        modifier = modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),          // リスト全体の内側余白
        verticalArrangement = Arrangement.spacedBy(8.dp) // 要素間の隙間
    ) {
        // 固定の見出し（DSLスコープなので普通の Composable とは書き方が違う）
        item {
            Text("ユーザー一覧", style = MaterialTheme.typography.titleLarge)
        }

        // リストを展開。key を渡すと「どの要素か」を Compose が追跡できる
        items(
            items = users,
            key = { user -> user.id }   // 安定したID（並べ替え・削除時に重要）
        ) { user ->
            UserRow(user)
        }

        // インデックスも欲しいとき
        itemsIndexed(users) { index, user ->
            Text("${index + 1}. ${user.name}")
        }
    }
}
```

`item { }`・`items(...) { }` は **`LazyListScope` のDSL** であり、`@Composable` 関数を直接並べる `Column` とは文法が異なる点に注意。

## 実務での使い方・定番パターン
- **`key` を必ず付ける**：`key = { it.id }` のように安定IDを渡すと、要素の追加・削除・並べ替えで「再利用すべき行」を Compose が正しく対応づけられる。アニメーションや状態（`remember`）のズレ防止に効く。
- **スクロール位置の取得・制御**は `rememberLazyListState()`：
  ```kotlin
  val listState = rememberLazyListState()
  val scope = rememberCoroutineScope()

  LazyColumn(state = listState) { /* items */ }

  // 例：先頭へ戻すボタン
  Button(onClick = { scope.launch { listState.animateScrollToItem(0) } }) {
      Text("先頭へ")
  }

  // 「ある程度スクロールしたら」を派生状態で
  val showFab by remember {
      derivedStateOf { listState.firstVisibleItemIndex > 5 }
  }
  ```
- **無限スクロール／追加読み込み**は、`listState.layoutInfo` の可視範囲を監視して末尾近くで次ページを取りに行く（取得自体は ViewModel / コルーチン側）。→ [async_coroutines.md](./async_coroutines.md)
- **セクション分け**は `item { ヘッダー }` と `items(group) { }` を交互に並べる。固定ヘッダーが要るなら `stickyHeader { }`。
- **グリッド**が必要なら `LazyVerticalGrid` / `LazyHorizontalGrid`（列数は `GridCells.Fixed(2)` など）。

## ハマりどころ / アンチパターン
- **大量データを普通の `Column` で回す**：`Column { list.forEach { Item(it) } }` は全要素を即描画する。スクロールするほどの件数になったら **必ず `LazyColumn`** に置き換える。これが最頻出の性能事故。
- **`key` 未指定**：デフォルトは位置ベースのため、先頭に1件挿入すると「全行がズレた別物」と認識され、行内の `remember` 状態やアニメが崩れる。**`key` は安定・一意（DBのidなど）**にする。リスト内のインデックスを key にしない。
- **`LazyColumn` を `verticalScroll` 付きの親に入れる**：高さが無限大になり「縦に無限スクロール可能なコンテナの中に無限の高さのリスト」で実行時クラッシュ。Lazy 系は自分でスクロールするので、外側に `verticalScroll` を重ねない。
- **`LazyColumn` に固定でない無限高さを与える**：`fillMaxSize()` や重み付き高さで領域を確定させる。`wrapContentHeight` で全件を測ろうとすると Lazy の利点が消える。
- **`items` ブロック内で重い処理（ソート・フィルタ）を毎回実行**：行の組み立て中に計算しない。並べ替え済みリストを `remember(key) { }` で用意しておく。
- **`item { }` の付け忘れ**：`LazyColumn { Text("x") }` はコンパイルエラー。中身は `item` / `items` DSL 経由で置く。

## 関連
[layout.md](./layout.md) / [state.md](./state.md) / [async_coroutines.md](./async_coroutines.md)
