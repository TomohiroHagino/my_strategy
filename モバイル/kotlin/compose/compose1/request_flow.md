# データの流れ・各部分は何を返すか（Jetpack Compose）

## ひとことで言うと
モバイルは「サーバのリクエスト層」ではなく、**状態→UIの一方向データフロー**で考える。**ユーザー操作 → State が変わる → 関連する `@Composable` が再コンポーズされる → UI が更新される**。データ取得は ViewModel が行い、結果を State（StateFlow）に流し込むと、それを購読している Composable が追従する。

## 全体の流れ（図）
```
ユーザー操作（クリック／入力）
   │
   ▼
[State 更新]   mutableStateOf の値変更 / ViewModel の _uiState.value = ...
   │           ※ 手で View を更新しない。状態を変えるだけ
   ▼
[再コンポーズ]  その State を読む @Composable だけ再実行
   │
   ▼
[UI 更新]      新しい状態に対応した UI に差し替わる
   │
   ▼
  画面（描画）

── データ取得フロー（横入り）──────────────
[Repository / API]  suspend 関数で取得（Retrofit など）
   │ データを返す
   ▼
[ViewModel]  取得結果を StateFlow に流す（_uiState.update { ... }）
   │ StateFlow を公開
   ▼
[Composable]  collectAsState() で購読 → 値が変われば再コンポーズ → UI 更新
```
一方向（状態 → UI）。Composable は状態を「読む」だけで、戻り矢印は無い。

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 役割 |
|---|---|---|---|
| **ユーザー操作** | クリック／入力 | イベント（ViewModel のメソッド呼び出し） | 状態変更のきっかけ |
| **Repository / API** | id / 条件 | **データ**（suspend の戻り値） | 外部からデータ取得 |
| **ViewModel** | 操作・取得結果 | **StateFlow（UI 状態）** | 状態を保持し UI へ公開 |
| **@Composable** | State（collectAsState） | **UI** | 状態から画面を組み立てる |
| **UI 要素** | — | 描画される UI そのもの | 運ばれる側 |

- **状態ホイスティング**：状態は Composable の外（ViewModel 等）に置き、Composable は値とコールバックを受け取るだけにすると再利用しやすい。
- **再コンポーズは局所的**：その状態を読む Composable のみが再実行される。

## コードで通して見る
```kotlin
// 1) ViewModel：操作を受け取り、Repository から取得して StateFlow を更新
class UserViewModel(private val repo: UserRepository) : ViewModel() {
  private val _uiState = MutableStateFlow(UiState(loading = true))
  val uiState: StateFlow<UiState> = _uiState.asStateFlow()   // 状態を公開

  fun load() {
    viewModelScope.launch {
      val user = repo.fetchUser()                  // API/Repository → データを返す
      _uiState.update { it.copy(loading = false, user = user) }  // State 更新
    }
  }
}

data class UiState(val loading: Boolean = false, val user: User? = null)
```

```kotlin
// 2) Composable：StateFlow を購読し、状態から UI を組み立てる
@Composable
fun UserScreen(viewModel: UserViewModel) {
  val state by viewModel.uiState.collectAsState()   // State を購読（変われば再コンポーズ）

  when {
    state.loading -> CircularProgressIndicator()
    state.user != null -> Column {
      Text(state.user!!.name)                        // 状態 → UI
      Button(onClick = { viewModel.load() }) {       // ユーザー操作 → ViewModel
        Text("再読み込み")
      }
    }
  }
}
```

## 実務での使い方・定番パターン
- **状態は ViewModel の StateFlow に集約**：画面状態を 1 つの `UiState` data class にまとめ、`collectAsState()` で購読する。→ [viewmodel_state.md](./viewmodel_state.md) / [state.md](./state.md)
- **取得は suspend → StateFlow → UI**：`viewModelScope.launch` で取得し、結果を状態に流す。loading/error も状態として持つ。→ [async_coroutines.md](./async_coroutines.md)
- **Composable に副作用を直書きしない**：起動時の取得は `LaunchedEffect` で。描画の中で API を叩かない。

## ハマりどころ / アンチパターン
- **`mutableStateOf` を `remember` せず使う**：再コンポーズのたびに初期化され状態が消える。`remember { mutableStateOf(...) }`。
- **再コンポーズで毎回 API を呼ぶ**：`LaunchedEffect(key)` を使わず本文で呼ぶと無限ループ・多重リクエスト。
- **`Flow` を直接 UI で扱う**：`collectAsState()` を通さないと状態として追従しない。
- **手続き的に View を更新しようとする**：Compose は宣言的。状態を変えてフレームワークに再描画させる。

## 関連
[state.md](./state.md) / [viewmodel_state.md](./viewmodel_state.md) / [async_coroutines.md](./async_coroutines.md) / [composables.md](./composables.md)
