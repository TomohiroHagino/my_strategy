# 状態（State）（Jetpack Compose）

## ひとことで言うと
**`@Composable` が参照する「変わりうる値」**のこと。`mutableStateOf(...)` で作った状態が変わると、その状態を読んでいる Composable だけが**自動で再コンポーズ**（再描画）される。手で `setText` などはしない。

## 役割・なぜ必要か
- Compose の UI は「**状態 → UI**」の一方向。状態を更新すれば画面がそれに追従する（宣言的 UI）。
- 命令的な View 操作（`findViewById().setText()`）を捨て、「いまの状態を見て、いまの見た目を返す関数」を書くだけにするための仕組み。
- ただの `var` ではダメ。Compose に「この値は観測対象」と知らせるため `mutableStateOf` で包み、`remember` で**再コンポーズをまたいで覚えさせる**必要がある。

## 基本の書き方（コード）
```kotlin
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember

@Composable
fun Counter() {
    // by デリゲートで count を素の Int のように読み書きできる
    var count by remember { mutableStateOf(0) }

    Column {
        Text("count = $count")
        Button(onClick = { count++ }) {   // 更新すると再コンポーズ
            Text("+1")
        }
    }
}
```
`by` を使わない場合は `.value` で読み書きする。
```kotlin
val count = remember { mutableStateOf(0) }
Text("count = ${count.value}")
Button(onClick = { count.value++ }) { Text("+1") }
```

## 実務での使い方・定番パターン
- **状態ホイスティング（state hoisting）**：状態を Composable の中に閉じ込めず、**呼び出し側（上位）へ持ち上げる**。下位の Composable は state を持たず「値 + onイベント」を受け取るだけの **stateless** にする。再利用・テスト・プレビューが効きやすくなる。
```kotlin
// stateless：自分では state を持たない
@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("count = $count") }
}

// stateful：state を持ち、下位へ流す（呼び出し側＝上位）
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    Counter(count = count, onIncrement = { count++ })
}
```
- **構成変更（画面回転・ダークモード切替・言語変更）でも保持**したい値は `rememberSaveable` を使う。`remember` は構成変更で破棄されるが、`rememberSaveable` は Bundle に退避して復元する。
```kotlin
// 画面を回転しても入力が消えない
var name by rememberSaveable { mutableStateOf("") }
TextField(value = name, onValueChange = { name = it })
```
- 画面をまたぐ／ロジックを伴う状態は Composable ではなく **ViewModel** へ寄せる。→ [viewmodel_state.md](./viewmodel_state.md)

## ハマりどころ / アンチパターン
- **`remember` を付け忘れる**：`var count by mutableStateOf(0)`（remember なし）だと、再コンポーズのたびに `0` に戻り続け、値が増えない。状態は必ず `remember`（または `rememberSaveable`）で囲む。
- **`by` デリゲートで import 忘れ**：`getValue` / `setValue`（`androidx.compose.runtime.getValue` / `setValue`）を import しないと `by` がコンパイルエラーになる。Android Studio の補完任せにすると入らないことがある。
- **状態をローカルに閉じ込めて再利用できない**：下位 Composable が自分で `remember` を持つと、親から制御できずテストもしづらい。**状態ホイスティング**で上位へ上げ、下位は stateless にする。
- **構成変更で消える**：`remember` だけだと画面回転で入力が飛ぶ。保持したい UI 状態は `rememberSaveable`。ただし保存できるのは Bundle に入る型のみ（独自クラスは `Saver` が必要）。
- **巨大オブジェクトや非同期結果を `rememberSaveable` に詰める**：Bundle は小さく軽い値向け。重い状態・サーバ由来データは ViewModel + StateFlow へ。→ [viewmodel_state.md](./viewmodel_state.md)
- **状態を直接 mutate して再コンポーズが走らない**：`mutableStateListOf` ではない普通の `MutableList` を中身だけ書き換えても再描画されない。状態は「新しい値の代入」または Compose 対応のコレクションで扱う。

## 関連
[composables.md](./composables.md) / [viewmodel_state.md](./viewmodel_state.md)
