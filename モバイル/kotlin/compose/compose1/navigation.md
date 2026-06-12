# ナビゲーション（Navigation Compose）

## ひとことで言うと
**Compose 上で画面遷移（バックスタック管理）を行う仕組み**。`NavController` が遷移を制御し、`NavHost` が「ルート文字列 → どの Composable を表示するか」の対応表を持ち、`navigate("route")` で遷移する。

## 役割・なぜ必要か
- Activity/Fragment を増やさず、**1つの画面（setContent）の中で複数のスクリーンを切り替える**ための公式ライブラリ（`androidx.navigation:navigation-compose`）。
- 「いまどの画面か」「戻るとどこへ」というバックスタックを自前管理せず、`NavController` に任せられる。
- 各画面を `composable("route")` で登録し、文字列ルートで指し示すルーティング方式。引数も URL ライクに渡せる。

## 基本の書き方（コード）
```kotlin
@Composable
fun AppNav() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = "home") {
        composable("home") {
            HomeScreen(onOpenDetail = { id ->
                navController.navigate("detail/$id")   // 引数付きで遷移
            })
        }
        // route に {id} を埋め込んで引数を受ける
        composable("detail/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailScreen(id = id, onBack = { navController.popBackStack() })
        }
    }
}
```
引数の型を指定したい場合は `arguments` を宣言する。
```kotlin
import androidx.navigation.NavType
import androidx.navigation.navArgument

composable(
    route = "detail/{id}",
    arguments = listOf(navArgument("id") { type = NavType.IntType }),
) { entry ->
    val id = entry.arguments?.getInt("id") ?: 0
    DetailScreen(id = id)
}
```

## 実務での使い方・定番パターン
- **戻る**：`navController.popBackStack()` または `navController.navigateUp()`。システムの戻るボタンは自動でバックスタックを辿る。
- **重複遷移を防ぐ**：ボトムナビ等で同じ画面を積み増さないよう `launchSingleTop` や `popUpTo` を使う。
```kotlin
navController.navigate("home") {
    popUpTo("home") { inclusive = true }  // home までスタックを掃除
    launchSingleTop = true                 // 同一ルートを多重に積まない
}
```
- **ルート文字列は定数化**：`object Routes { const val HOME = "home" }` のように一元管理し、タイプミスを防ぐ。
- **ViewModel と組む**：画面ごとの状態・ロジックは ViewModel に持たせ、`composable` 内で取得する。引数（id 等）は ViewModel の初期化に渡す。→ [viewmodel_state.md](./viewmodel_state.md)
- **型安全ルート（新しめ）**：Navigation 2.8 以降、`@Serializable` なデータクラスを destination にできる **Type-Safe Navigation**（`navigation-compose` のジェネリック `composable<T>` / `navigate(T)`）がある。文字列ルートの脱字や引数の型ミスをコンパイル時に防げるので、新規実装ではこちらを検討する。

## ハマりどころ / アンチパターン
- **`NavHost` の設定漏れ**：`startDestination` のルートを `composable(...)` で登録し忘れると即クラッシュ（該当 destination なし）。ルート名は登録側と遷移側で完全一致が必要。
- **route 文字列のタイプミス**：`"detail/{id}"` と `navigate("detail/$id")` の綴りズレで遷移先が見つからない。定数化＋型安全ルートで回避する。
- **引数の渡し方ミス**：route の `{id}` プレースホルダ名と `getString("id")`/`navArgument("id")` のキーが一致していないと取り出せず null。`NavType` の不一致（String を Int で取る等）も落とし穴。
- **`rememberNavController()` を使い回さない**：再コンポーズのたびに新しい NavController を作るとバックスタックが壊れる。必ず `remember` 系で1つを保持する。
- **画面状態を NavController に持たせようとする**：NavController は遷移管理が責務。画面の業務状態は ViewModel/State へ。→ [viewmodel_state.md](./viewmodel_state.md)
- **戻る挙動が意図と違う**：多重 navigate でスタックが積み上がる。`launchSingleTop` / `popUpTo` を適切に設定する。

## 関連
[viewmodel_state.md](./viewmodel_state.md) / [state.md](./state.md)
