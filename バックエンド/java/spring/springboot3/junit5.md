# JUnit 5（Spring Boot 3）

## ひとことで言うと
Java のテスト実行基盤（テストフレームワーク）。`@Test` を付けたメソッドを発見・実行し、結果を判定する。Spring Boot 3 のテストは全てこの上に乗る土台で、`@WebMvcTest` などの Spring 用アノテーションも JUnit 5 の拡張機構（`@ExtendWith`）として動く。

## 役割・なぜ必要か
- 「どのメソッドがテストか」「いつ前処理・後処理を走らせるか」を**アノテーションで宣言**し、実行順序やライフサイクルを管理する。
- JUnit 4 から刷新され、`@ExtendWith` による拡張モデルになった。Spring（`SpringExtension`）・Mockito（`MockitoExtension`）はこの拡張として差し込まれる。
- アサーション（`assertEquals` 等）も持つが、実務では読みやすい [AssertJ](./assertj.md) を併用するのが定番。JUnit はあくまで**実行の枠組み**を担う。

## 基本の書き方（コード）
```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    Calculator calc;

    @BeforeEach            // 各テストの前に毎回実行（状態を初期化）
    void setUp() {
        calc = new Calculator();
    }

    @Test
    @DisplayName("1 + 2 は 3 になる")   // レポートに出る読みやすい名前
    void add() {
        assertEquals(3, calc.add(1, 2));
    }

    @Test
    void ゼロ除算は例外() {
        // 例外が投げられることを検証（型と発生を同時に確認）
        assertThrows(ArithmeticException.class, () -> calc.div(1, 0));
    }
}
```
```java
// ライフサイクル：クラス全体で1回 vs 各テストで毎回
class LifecycleTest {
    @BeforeAll static void beforeAll() {}   // クラス全体の前に1回（static）
    @AfterAll  static void afterAll()  {}   // クラス全体の後に1回（static）
    @BeforeEach void beforeEach() {}        // 各テストの前に毎回
    @AfterEach  void afterEach()  {}        // 各テストの後に毎回
}
```
```java
// パラメータ化テスト：同じロジックを複数入力で回す
class ParamTest {
    @ParameterizedTest
    @ValueSource(ints = {2, 4, 100})
    void すべて偶数(int n) {
        assertEquals(0, n % 2);
    }

    @ParameterizedTest
    @CsvSource({"1, 2, 3", "10, 5, 15"})   // 入力と期待値を組で渡す
    void 加算(int a, int b, int expected) {
        assertEquals(expected, a + b);
    }
}
```
```java
// @Nested：文脈ごとにテストをグループ化（given-when 構造を表現）
class OrderTest {
    @Nested
    @DisplayName("在庫があるとき")
    class WhenInStock {
        @Test void 注文は成功する() { /* ... */ }
    }

    @Nested
    @DisplayName("在庫が無いとき")
    class WhenOutOfStock {
        @Test void 注文は失敗する() { /* ... */ }
    }
}
```

## 実務での使い方・定番パターン
- **Spring Boot Starter Test に同梱**：`spring-boot-starter-test` を入れれば JUnit 5・Mockito・AssertJ がまとめて入る。個別依存は基本不要。
- **`@ExtendWith` で拡張を差し込む**：Spring を使わない単体は `@ExtendWith(MockitoExtension.class)`、Spring コンテキストが要るスライステストは `@WebMvcTest` 等が内部で `SpringExtension` を有効化する。
- **`@DisplayName` で日本語の仕様名**：「在庫不足なら例外を投げる」のように振る舞いを書くと、CI のレポートがそのまま仕様書になる。
- **`@ParameterizedTest` で重複排除**：境界値・複数ケースは1メソッドにまとめ、テストコードの DRY を保つ。
- **`@Nested` で文脈を整理**：「〜のとき」をネストクラスで表現すると、前提条件ごとの `@BeforeEach` を分けられて読みやすい。

## ハマりどころ / アンチパターン
- **JUnit 4 の import を混ぜる**：`org.junit.Test`（4系）と `org.junit.jupiter.api.Test`（5系）は別物。4系を import するとテストが**発見されず無言でスキップ**される。`jupiter` を使う。
- **`@BeforeAll`/`@AfterAll` を非 static にする**：既定のライフサイクル（クラスごとにインスタンス生成）では static でないと動かない。`@TestInstance(PER_CLASS)` を付ければ非 static も可。
- **テスト間の状態リーク**：フィールドを `@BeforeEach` で初期化せず使い回すと順序依存で落ちる。各テストで初期化する。
- **`assertThrows` の戻り値を捨てる**：例外メッセージまで検証したいなら戻り値の例外オブジェクトを受け取り `getMessage()` を確認する。
- **private なテストメソッド**：JUnit 5 はパッケージプライベート以上で発見する。`private` は実行されない（public は不要だが private は不可）。

## 関連
[testing.md](./testing.md) / [mockito.md](./mockito.md) / [assertj.md](./assertj.md)
