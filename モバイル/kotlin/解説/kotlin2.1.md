# Kotlin 2.1（言語解説）

## ひとことで言うと
JetBrains による静的型付け・null 安全の言語で、JVM / Native / JS / Wasm にコンパイルできる。2.0 で新コンパイラ K2 が既定になり、2.1 はそれを土台に守られた `when` の網羅やガード条件などの言語機能を加えたバージョン。Android 開発の公式言語であり、コルーチンによる構造化並行が中心的な道具になる。

## このバージョンの位置づけ（リリース / サポート / どこで使うか）
- Kotlin 2.0 で K2 コンパイラがデフォルト化した。これは型推論・解析・補完を作り直したもので、ビルドが速くなり挙動が一貫した。2.1 はその上での言語拡張リリース（2024 年後半）。
- ターゲットは複数ある。JVM（Android / サーバ）が主流、Kotlin/Native で iOS バイナリ、Kotlin/JS と Kotlin/Wasm でブラウザ。共通ロジックを共有する Kotlin Multiplatform（KMP）の基盤でもある。
- 主用途は Android ネイティブアプリ、サーバ（Ktor / Spring Boot）、KMP。

```bash
kotlinc -version          # コンパイラのバージョン
kotlin script.main.kts    # スクリプト実行
./gradlew build           # 通常は Gradle 経由でビルド
```

## 言語の基本（文法の要点）
変数は `val`（再代入不可）と `var`（可変）。型は推論されるが明示もできる。

```kotlin
val name = "Kotlin"      // 推論で String、再代入不可
var count = 0            // 可変
val pi: Double = 3.14    // 明示型
```

関数は `fun`。単一式なら `=` で書ける。デフォルト引数と名前付き引数を持つ。

```kotlin
fun greet(name: String, greeting: String = "Hello"): String =
    "$greeting, $name!"

greet("A")                      // greeting はデフォルト
greet(name = "B", greeting = "Hi")  // 名前付き
```

`if` と `when` は式（値を返す）。`when` は値の分岐にも条件の分岐にも使える。

```kotlin
val grade = when (score) {
    in 90..100 -> "A"
    in 70..89 -> "B"
    else -> "C"
}
```

文字列テンプレート `$変数` / `${式}`、範囲 `1..10`、`for (x in list)` などが基本構文。

## この言語の核心概念（他言語と違う・必ず押さえる）
Kotlin は「Java の問題を型で直す」言語。文法の前に、この7点が Kotlin らしさの中心。

### null 安全（Java との最大の差）
①何か：型に null 可否を埋め込み、`?` 付きだけが null を持てる。`?.`（null なら飛ばす）・`?:`（エルビス、null のときの代替）・`!!`（null でないと断言）が道具。`!!` を使った箇所が実際に null だと `NullPointerException` になる。

②具体コード：
```kotlin
var nonNull: String = "ok"     // null 不可
var maybe: String? = null      // null 可

val len = maybe?.length ?: 0   // null なら 0
val forced = maybe!!           // null なら例外（危険）
```

③他言語と違う点/つまずき：Java はあらゆる参照が null になりうり、null チェックは規律でしかない。Kotlin は型が強制する。最大の落とし穴は Java 相互運用で、Java 由来の型は null 性が不明な「プラットフォーム型」（`String!`）になり、Kotlin の null チェックが効かないまま `!!` 相当の扱いになる。`!!` の乱用は Kotlin で書きながら Java の危険を呼び戻す行為。

### `val` / `var`（不変参照を推奨）
①何か：`val` は再代入できない参照、`var` は再代入できる参照。Kotlin は既定で `val` を使い、`var` は必要なときだけにする文化。

②具体コード：
```kotlin
val name = "Kotlin"        // 再代入不可
var count = 0              // 再代入可
count += 1
val list = mutableListOf(1)
list.add(2)               // ← val でも中身は変えられる
```

③他言語と違う点/つまずき：`val` は「参照の固定」であって「中身の不変」ではない。`val list = mutableListOf(...)` の要素は追加・変更できる。本当に変えさせたくないなら読み取り専用の `listOf` を使う。ここを「val なら不変」と勘違いしやすい。

### データクラス（`equals` / `copy` 自動生成）
①何か：`data class` を付けると、`equals` / `hashCode` / `toString` / `copy` / 分解用 `componentN` をコンパイラが自動生成する、値の入れ物のための仕組み。

②具体コード：
```kotlin
data class User(val id: Int, val name: String)
val u = User(1, "A")
val u2 = u.copy(name = "B")        // 一部だけ変えた新インスタンス
println(u == User(1, "A"))         // true（構造的等価）
val (id, name) = u                 // 分解宣言
```

③他言語と違う点/つまずき：Java では `equals`/`hashCode`/`toString` を手書き（または Lombok）する。Kotlin は標準機能。等価判定が「同一インスタンスか」ではなく「フィールドが等しいか（構造的等価）」になる点が、Java のデフォルト挙動と逆向きで戸惑いやすい。`copy` はコンストラクタ引数（プロパティ）だけが対象。

### 拡張関数（既存クラスに後付け）
①何か：既存クラスを継承・改変せず、外からメソッドを足したように呼べる関数。レシーバ型に対して `fun レシーバ型.名前()` と書く。

②具体コード：
```kotlin
fun String.shout(): String = uppercase() + "!"
"hello".shout()        // "HELLO!"

fun List<Int>.secondOrNull(): Int? = getOrNull(1)
listOf(10, 20).secondOrNull()   // 20
```

③他言語と違う点/つまずき：実体はレシーバを第一引数に取る静的関数で、クラス本体を書き換えるわけではない。よって `private` メンバには触れず、ディスパッチも静的（オーバーライドされない）。標準ライブラリのほとんど（`let`/`map` 等）がこの仕組みで作られている。

### スマートキャスト（`is` の後に自動キャスト）
①何か：`is` で型を確認すると、それ以降そのブロックでは確認済みの型として明示キャストなしで扱える。null チェック後の非 null 化も同様。

②具体コード：
```kotlin
fun describe(x: Any): String {
    if (x is String) return "len=${x.length}"  // 以降 x は String
    return "other"
}
fun lenOf(s: String?): Int {
    if (s == null) return 0
    return s.length            // null チェック済みなので非 null 扱い
}
```

③他言語と違う点/つまずき：Java は `instanceof` の後でも明示キャストが要る（Java 16+ のパターン変数で改善）。Kotlin はコンパイラが追跡して自動でキャストする。ただし `var` のメンバプロパティなど「途中で変わりうる値」はスマートキャストが効かない（間に別スレッドが変える可能性があるため）。K2 コンパイラで効く範囲が広がった。

### コルーチン / `suspend`（スレッドと違う構造化並行）
①何か：`suspend` 関数は途中で中断・再開でき、待っている間スレッドをブロックしない。`launch`/`async` で起動し、`coroutineScope` の構造化並行で子のライフサイクルを束ねる。連続値は `Flow`。

②具体コード：
```kotlin
import kotlinx.coroutines.*

suspend fun fetch(): String { delay(1000); return "data" }  // スレッドを止めず待つ

suspend fun loadAll() = coroutineScope {
    val a = async { fetch() }      // 並行起動
    val b = async { fetch() }
    a.await() + b.await()          // 両方そろうまで待つ。1つ失敗で兄弟もキャンセル
}
```

③他言語と違う点/つまずき：スレッドは OS 資源で生成コストが重く、ブロックすると占有される。コルーチンは軽量で、`delay` 中はスレッドを他の仕事に回せる。構造化並行により `coroutineScope` を抜けるまでに子が必ず完了/キャンセルされリークしない。逆に `GlobalScope.launch` で起動するとこの保護から外れてリークする。`Flow` はコールドで、`collect` するまで何も流れない。

### sealed class / `when` 網羅
①何か：`sealed class` / `sealed interface` はサブタイプを同一モジュールに閉じ、`when` で全サブタイプを分岐するとコンパイラが網羅を検査して `else` を不要にする。状態の表現に向く。

②具体コード：
```kotlin
sealed interface Result
data class Ok(val value: Int) : Result
data class Err(val msg: String) : Result

fun handle(r: Result) = when (r) {
    is Ok -> r.value          // スマートキャストで Ok として扱える
    is Err -> -1
    // 全サブタイプを書けば else 不要、追加で漏れればコンパイルエラー
}
```

③他言語と違う点/つまずき：通常の `when (式)` には `else` が必須だが、`sealed` 型に対する `when` は網羅していれば `else` を省ける。後でサブタイプを足したとき、網羅していない `when` がコンパイルエラーになって直し漏れを防げる。これが「`else -> throw ...` で握りつぶす」Java 的な書き方との差。

## 型・データモデル
- 基本型は `Int` / `Long` / `Double` / `Boolean` / `Char` / `String`。コレクションは読み取り専用（`List` / `Map` / `Set`）と可変（`MutableList` 等）が型で分かれている。
- `Any` が頂点、`Nothing` が底の型。ジェネリクスを持ち、`out`（共変）/ `in`（反変）で変位を指定できる。

null 安全は型システムの核。`?` 付きが null 許容で、未付きは null を代入できない。

```kotlin
var nonNull: String = "ok"      // null 不可
var maybe: String? = null       // null 可
val len = maybe?.length ?: 0    // ?. と ?: エルビス演算子
val forced = maybe!!            // !! は null でないと断言（外すと例外）
```

`data class` は等価判定・`toString`・`copy` を自動生成する値の入れ物。

```kotlin
data class User(val id: Int, val name: String)
val u = User(1, "A")
val u2 = u.copy(name = "B")     // 一部だけ変えた新インスタンス
```

`sealed class` / `sealed interface` はサブタイプを閉じ、`when` で網羅できる。

```kotlin
sealed interface Result
data class Ok(val value: Int) : Result
data class Err(val msg: String) : Result

fun handle(r: Result) = when (r) {
    is Ok -> r.value          // スマートキャストで Ok として扱える
    is Err -> -1
    // 全サブタイプを書けば else 不要
}
```

## この言語らしさ / 特徴的な機能
拡張関数は既存クラスを継承せずにメソッドを足せる。

```kotlin
fun String.shout(): String = uppercase() + "!"
"hello".shout()   // "HELLO!"
```

スマートキャストは `is` で型を確認した後、その型として自動的に扱える（K2 でより広い場面に効くようになった）。

```kotlin
fun describe(x: Any): String {
    if (x is String) return "len=${x.length}"  // 以降 x は String
    return "other"
}
```

スコープ関数（`let` / `run` / `apply` / `also` / `with`）でオブジェクト処理を簡潔に書く。

```kotlin
val config = StringBuilder().apply {
    append("a")
    append("b")     // this が StringBuilder、append を直接呼べる
}
```

`object` でシングルトン、`companion object` でクラス付随のメンバ、`inline` 関数で高階関数のオーバーヘッド削減、と道具が揃う。

## 並行・非同期
非同期はコルーチンで書く。`suspend` 関数は中断・再開でき、スレッドをブロックしない。

```kotlin
import kotlinx.coroutines.*

suspend fun fetch(): String {
    delay(1000)        // スレッドを止めずに 1 秒待つ
    return "data"
}
```

構造化並行が要。`coroutineScope` の中で起動した子は、スコープを抜けるまでに全て完了し、1 つが失敗すると兄弟もキャンセルされる。リークしない。

```kotlin
suspend fun loadAll() = coroutineScope {
    val a = async { fetch() }    // 並行起動
    val b = async { fetch() }
    a.await() + b.await()        // 両方そろうまで待つ
}
```

連続値は `Flow`（コールド・非同期ストリーム）で扱う。

```kotlin
fun numbers(): Flow<Int> = flow {
    for (i in 1..3) { emit(i); delay(100) }
}
// collect するまで何も流れない（コールド）
```

## 標準ライブラリ / ツールチェーン
- 標準ライブラリにコレクション操作（`map` / `filter` / `fold` / `groupBy` 等）、シーケンス（遅延評価）、`Result` 型、範囲、文字列ユーティリティが揃う。
- コルーチン（`kotlinx-coroutines`）と直列化（`kotlinx-serialization`）は公式ライブラリ（標準ライブラリとは別の依存）。
- ビルドは Gradle（Kotlin DSL の `build.gradle.kts` が推奨）。Android では Android Gradle Plugin が絡む。整形・静的解析は ktlint / detekt を併用するのが一般的。

```kotlin
// build.gradle.kts（抜粋）
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0")
}
```

## このバージョンの新機能・トピック
- K2 コンパイラ（2.0 で既定）：解析エンジンを刷新。型推論やスマートキャストが安定・高速になり、マルチプラットフォームでも挙動が揃った。2.1 はこの上での言語機能追加。
- 2.1 の `when` のガード条件：分岐に追加の条件を `if` で付けられる。

```kotlin
val msg = when (r) {
    is Ok if r.value > 0 -> "positive"   // ガード条件
    is Ok -> "non-positive"
    is Err -> "error"
}
```

- 2.1 では非ローカルな `break` / `continue`（インライン関数のラムダ内からループを抜ける）や、複数ドルの文字列補間など、書き味を整える機能も入った。
- Kotlin/Wasm のサポートが進み、ブラウザ向けターゲットとしての実用度が上がっている。

## ハマりどころ
- `!!`（null 断言）の乱用：null でないと断言した値が null だと `NullPointerException`。`?.` と `?:` で逃がすのが基本。
- `val` は再代入不可だが不変ではない：`val list = mutableListOf(1)` の中身は変更できる。本当に不変なら読み取り専用の `listOf` を使う。
- 読み取り専用コレクションは「読み取り専用ビュー」であって不変保証ではない。元が可変なら別経路で変わりうる。
- コルーチンを `GlobalScope` で起動するとライフサイクルから外れてリークする。`coroutineScope` や適切なスコープを使う。
- `Flow` はコールド：`collect` するまで動かない。起動済みと勘違いしやすい。
- Java 相互運用での null：Java 由来の型は null 性が不明（プラットフォーム型）で、Kotlin 側で null チェックが効かないことがある。

## 関連
- [../compose/](../compose/) … Android の現行宣言的 UI フレームワーク Jetpack Compose の版別リファレンス。
- [../android_studio.md](../android_studio.md) … Kotlin ネイティブ開発の IDE である Android Studio の解説。
- 同フォルダの [README.md](../README.md) … Kotlin 言語の概要・強み弱み・エコシステム。
