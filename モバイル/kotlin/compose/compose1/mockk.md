# MockK（Jetpack Compose）

## ひとことで言うと
Kotlin 用のモックライブラリ。テスト対象が依存するクラス/インターフェースの**偽物（モック）**を `mockk()` で作り、戻り値を `every { … } returns …` で固定し、呼ばれ方を `verify { … }` で検証する。`suspend` 関数には `coEvery` / `coVerify`、トップレベル関数や拡張関数の差し替えは `mockkStatic` を使う。Kotlin の null 安全・ラムダ DSL に合わせて設計されている。

## 役割・なぜ必要か
- ViewModel/UseCase の unit テストで、依存する Repository や API を**本物を叩かずにモック化**する。
- 「成功」「失敗」「空」などの戻り値を**テスト側から作れる**ので、`StateFlow` の Loading/Success/Error 分岐を確実に通せる。
- 依存先メソッドが**正しい引数で・正しい回数呼ばれたか**を `verify` で保証できる。
- `suspend`（コルーチン）にネイティブ対応（`coEvery`/`coVerify`）しており、Kotlin の非同期コードをそのままモックできる。

## 基本の書き方（コード）
```kotlin
// build.gradle.kts
dependencies {
    testImplementation("io.mockk:mockk:1.13.12")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
}
```
```kotlin
// 対象コード（例）
interface UserRepository {
    suspend fun fetchName(id: Int): String
}

class Greeter(private val repo: UserRepository) {
    suspend fun greet(id: Int): String = "Hello, ${repo.fetchName(id)}"
}
```
```kotlin
// GreeterTest.kt
import io.mockk.*
import kotlinx.coroutines.test.runTest
import kotlin.test.*

class GreeterTest {
    @Test
    fun `greet はリポジトリの名前で挨拶を返す`() = runTest {
        // Arrange
        val repo = mockk<UserRepository>()
        // suspend 関数は coEvery で固定
        coEvery { repo.fetchName(1) } returns "Taro"
        val greeter = Greeter(repo)

        // Act
        val result = greeter.greet(1)

        // Assert
        assertEquals("Hello, Taro", result)
        // suspend の呼び出し検証は coVerify
        coVerify(exactly = 1) { repo.fetchName(1) }
    }

    @Test
    fun `失敗を再現する`() = runTest {
        val repo = mockk<UserRepository>()
        coEvery { repo.fetchName(any()) } throws RuntimeException("network")
        val greeter = Greeter(repo)
        assertFailsWith<RuntimeException> { greeter.greet(1) }
    }
}
```
```kotlin
// 同期メソッドは every / verify
val calc = mockk<Calculator>()
every { calc.add(1, 2) } returns 3
verify { calc.add(1, 2) }

// 戻り値を使わない（Unit を返す）モックは relaxed か just Runs
val logger = mockk<Logger>(relaxed = true)   // 全メソッドにデフォルト戻り値
every { logger.log(any()) } just Runs        // Unit を返すと明示
```

## 実務での使い方・定番パターン
- **`every` と `coEvery` の使い分け**：通常関数は `every { … } returns …`、`suspend` 関数は `coEvery`。検証も同様に `verify` / `coVerify`。混同すると `no answer found` 例外になる。
- **`relaxed = true`**：戻り値をいちいち定義したくない依存（ロガー等）に。`relaxedUnitFun = true` で Unit 関数だけ緩くもできる。
- **引数マッチャ `any()` / `eq()` / `match { … }`**：`any()` は何でも一致、`match` で条件指定。`slot` でキャプチャして実引数をアサートできる。
- **`verify(exactly = n)` / `verifyOrder` / `verifySequence`**：回数・順序を検証。`wasNot Called` で「呼ばれていない」も確認。
- **ViewModel テスト**：Repository をモック注入し、`StateFlow` を `runTest` + `coEvery` で Loading→Success の遷移を検証する。→ [viewmodel_state.md](./viewmodel_state.md)
- **`@MockK` + `MockKAnnotations.init(this)`**：複数モックを `@MockK lateinit var` で宣言し `@Before` で一括初期化する書き方も定番。
- **`mockkStatic` / `mockkObject`**：トップレベル関数・拡張関数・`object` を差し替えるとき。使用後は `unmockkAll()` で戻す。

## ハマりどころ
- **`suspend` に `every` を使う**：`every { repo.fetchName(1) }` だと `coEvery` を促す例外。suspend は必ず `coEvery`/`coVerify`。
- **`mockkStatic`/`mockkObject` の後始末忘れ**：差し替えたまま他テストに漏れて不安定化。`@After` で `unmockkAll()`、または `unmockkStatic` する。
- **Unit を返す関数の未定義**：戻り値が `Unit` でも `every { … } just Runs` か `relaxed` を指定しないと `no answer found` で落ちる。
- **コルーチンの待ち忘れ**：`runTest` で囲まないと suspend がそのまま進まずアサートが空振り。`kotlinx-coroutines-test` の `runTest` を使う。
- **final クラスのモック**：MockK は inline mock で final もモックできるが、それでも依存は**interface に対して**切る方が安定しテストしやすい。
- **検証し過ぎ**：内部呼び出し回数まで縛ると、リファクタで赤くなる「もろい」テストに。境界（Repository/API）だけ検証する。
- **`relaxed` の事故**：何でもデフォルト値を返すため、本来固定すべき戻り値の指定漏れに気づきにくい。重要な依存は明示的に `every` で固定する。

## 関連
[viewmodel_state.md](./viewmodel_state.md) / [async_coroutines.md](./async_coroutines.md) / [state.md](./state.md)
