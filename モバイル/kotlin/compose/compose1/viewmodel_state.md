# ViewModel / StateFlow（Jetpack Compose）

## ひとことで言うと
**`ViewModel`** は「画面（の状態とロジック）を持つ箱」で、**画面回転などの構成変更をまたいで生き残る**。その中の **`StateFlow`** が「現在のUI状態」を保持し、Compose は `collectAsStateWithLifecycle()` でそれを購読して再コンポーズする。

## 役割・なぜ必要か
- `remember { mutableStateOf(...) }` は **Composable のスコープ**の状態。画面回転（構成変更）で Activity が作り直されると **消える**。「入力途中・読み込み済みデータ・選択状態」を失いたくないものは、ここでは持てない。
- `ViewModel`（`androidx.lifecycle`）は構成変更を生き延びるため、**残したい状態の置き場**になる。Composable は「描画役」、ViewModel は「状態とロジックの保持役」という **MVVM** の分担。
- 状態の流れを **`StateFlow`（常に最新値を1つ持つ Flow）** で一方向に流すと、「状態が変わる → UIが追従する」という Compose の核と素直につながる。

## 基本の書き方（コード）
```kotlin
// UI状態は1つのimmutableなデータクラスにまとめる（部分更新は copy で新規生成）
data class UserUiState(
    val isLoading: Boolean = false,
    val users: List<User> = emptyList(),
    val error: String? = null
)

class UserViewModel(
    private val repo: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow() // 外へは読み取り専用で公開

    fun load() {
        _uiState.update { it.copy(isLoading = true, error = null) }
        viewModelScope.launch {                    // ViewModel破棄で自動キャンセル
            runCatching { repo.fetchUsers() }
                .onSuccess { list -> _uiState.update { it.copy(isLoading = false, users = list) } }
                .onFailure { e -> _uiState.update { it.copy(isLoading = false, error = e.message) } }
        }
    }
}
```

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = viewModel()) {
    // ライフサイクルに応じて collect を止める版（推奨）
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) { viewModel.load() }      // 初回だけ呼ぶ

    when {
        state.isLoading      -> CircularProgressIndicator()
        state.error != null  -> Text("失敗: ${state.error}")
        else                 -> UserList(state.users)
    }
}
```

## 実務での使い方・定番パターン
- **UI状態は1つの `data class` に集約**して `StateFlow<UiState>` で持つ。フィールドが増えても購読側は1本でよく、`copy` で部分更新する（immutable）。
- **公開は `asStateFlow()` で読み取り専用**に。`MutableStateFlow` を外へ出すと画面側から書き換えられ、一方向データフローが崩れる。
- **収集は `collectAsStateWithLifecycle()`** を第一選択（`androidx.lifecycle:lifecycle-runtime-compose`）。画面が止まっている間は購読を止め、無駄な処理を避ける。素の `collectAsState()` はバックグラウンドでも流し続ける。
- **イベント（1回きりの副作用：スナックバー・画面遷移）** は状態に混ぜず、`Channel` / `SharedFlow` で別に流すと「回転で再表示」事故を避けやすい。
- **生成は `viewModel()`**（`androidx.lifecycle.viewmodel.compose`）。依存注入が要るなら Hilt の `hiltViewModel()` や `ViewModelProvider.Factory`。

## ハマりどころ / アンチパターン
- **回転で消したくない状態を `remember` に置く**：`remember { mutableStateOf(...) }` は構成変更で消える。**残すべき状態は ViewModel へ**。逆に「展開/折りたたみ」程度の一時的UI状態は `remember`（＋必要なら `rememberSaveable`）で十分で、何でも ViewModel に入れない。→ [state.md](./state.md)
- **`StateFlow` の collect をライフサイクル無視で行う**：素の `collectAsState()` だと非表示中も収集が続く。基本は `collectAsStateWithLifecycle()`。
- **`MutableStateFlow` を public 公開**：UIから直接書けてしまい、状態変更の出所が散らかる。必ず `asStateFlow()`。
- **`ViewModel` に Context / View / Activity を持たせる**：構成変更を生き延びる ViewModel が古い Activity を握り続け **メモリリーク**。`Application` 由来が要るなら `AndroidViewModel`。
- **`copy` を忘れて中身を直接 mutate**：`StateFlow` は参照が同じだと「変化なし」とみなし再コンポーズされない。新インスタンスを `update { it.copy(...) }` で流す。
- **副作用（読み込み）を `@Composable` 本体に直書き**：再コンポーズのたびに走る。`LaunchedEffect` か ViewModel 側で起動する。→ [async_coroutines.md](./async_coroutines.md)

## 関連
[state.md](./state.md) / [async_coroutines.md](./async_coroutines.md) / [lists.md](./lists.md)
