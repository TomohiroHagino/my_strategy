# 非同期（コルーチン / LaunchedEffect）（Jetpack Compose）

## ひとことで言うと
通信・DB・待ち時間のある処理は **コルーチン（`suspend` 関数）** で書く。Compose の中でそれを安全に起動するのが **`LaunchedEffect` などの Effect API**。`@Composable` 関数本体から `suspend` を直接は呼べない。

## 役割・なぜ必要か
- `@Composable` 関数は **状態が変わるたびに何度でも再実行（再コンポーズ）される**。そこに通信などの副作用を直書きすると、再コンポーズのたびに走って多重リクエストやちらつきの原因になる。
- そこで Compose は「**いつ・どのキーで起動し、いつキャンセルするか**」を管理する **Effect API** を用意している。`LaunchedEffect` はその代表で、**コンポジションに入ると起動し、退場（破棄）すると自動でキャンセル**する。
- `suspend` は「中断・再開できる関数」で、メインスレッドをブロックせずに待てる。これにより重い処理中もUIが固まらない。

## 基本の書き方（コード）
```kotlin
@Composable
fun ArticleScreen(id: String) {
    var article by remember { mutableStateOf<Article?>(null) }

    // key が変わると「前の起動をキャンセル → 新しく起動」。退場時もキャンセルされる
    LaunchedEffect(id) {
        article = repository.fetchArticle(id)   // suspend 関数を Effect の中で呼ぶ
    }

    article?.let { Text(it.title) } ?: CircularProgressIndicator()
}
```

```kotlin
@Composable
fun SaveButton() {
    // クリックなど「イベント起点」で起動したいときのスコープ
    val scope = rememberCoroutineScope()
    val snackbar = remember { SnackbarHostState() }

    Button(onClick = {
        scope.launch {                           // ここなら suspend を呼べる
            repository.save()
            snackbar.showSnackbar("保存しました")
        }
    }) { Text("保存") }
}
```

```kotlin
// ViewModel 側：画面の寿命に紐づくスコープ
class ArticleViewModel : ViewModel() {
    fun refresh() = viewModelScope.launch {     // ViewModel破棄で自動キャンセル
        repository.refresh()
    }
}
```

## 実務での使い方・定番パターン
- **画面表示で1回だけ走らせたい**：`LaunchedEffect(Unit) { ... }`（キーが変わらないので再起動しない）。初回ロードや初期化に使う。
- **引数に応じて再取得**：`LaunchedEffect(id)` のように **依存する値を key にする**。`id` が変われば自動で古い処理をキャンセルして取り直す。
- **クリック等のイベント起点**：`rememberCoroutineScope()` の `scope.launch { }`。`LaunchedEffect` は「コンポジションのライフサイクル」起点なので、ボタン押下のような単発イベントにはこちらが適切。
- **重い処理・通信の本体は ViewModel の `viewModelScope`** に置くのが基本。Composable 側は状態を購読するだけにすると、責務が分かれてテストしやすい。→ [viewmodel_state.md](./viewmodel_state.md)
- **その他の Effect**：購読の開始/解除など後始末が要るなら `DisposableEffect`、最新値を捕まえたいだけなら `rememberUpdatedState`、状態の派生計算は `derivedStateOf`。

## ハマりどころ / アンチパターン
- **`@Composable` 本体で `suspend` を直接呼ぶ**：コンパイルエラー（`@Composable` は suspend を呼べない）。**必ず `LaunchedEffect` / `rememberCoroutineScope` などの Effect API 経由**で起動する。これが最頻出の入口エラー。
- **`@Composable` 本体に副作用を直書き**（`val x = repository.fetch()` を本体に置く等）：再コンポーズのたびに走り、多重リクエスト・ちらつき・無限ループの原因。副作用は Effect の中へ。
- **`LaunchedEffect` の key を `Unit` に固定したまま、本当は変わる値に依存している**：`id` が変わっても再取得されず古いデータが残る。**依存値を key に列挙**する（`LaunchedEffect(id, filter)`）。
- **クリックハンドラに `LaunchedEffect` を使う**：`LaunchedEffect` は表示時起点で、ボタン押下の単発処理には合わない。`rememberCoroutineScope().launch` を使う。
- **スコープを取り違えてリーク**：画面を離れても続けたくない処理を `GlobalScope` で起動しない。画面寿命なら `viewModelScope` / `LaunchedEffect`、UI操作なら `rememberCoroutineScope`。これらは破棄時に自動キャンセルされる。
- **`withContext(Dispatchers.IO)` で結果を受けた後にUI更新を別スレッドのまま行う**：状態更新はメイン側で。IO はあくまで重い処理の実行に限定する。

## 関連
[viewmodel_state.md](./viewmodel_state.md) / [state.md](./state.md) / [lists.md](./lists.md)
