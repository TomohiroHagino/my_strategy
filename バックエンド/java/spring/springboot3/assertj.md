# AssertJ（Spring Boot 3）

## ひとことで言うと
**流暢（fluent）なアサーションライブラリ**。`assertThat(実際).isEqualTo(期待)` のように左から右へ自然に読める形で検証を書く。JUnit 標準の `assertEquals` より可読性が高く、コレクション・例外・オブジェクトの検証メソッドが豊富。Spring Boot Test に同梱され、実務の検証の主役になる。

## 役割・なぜ必要か
- `assertEquals(期待, 実際)` は引数の順番を間違えやすく、失敗メッセージも素っ気ない。AssertJ は **`assertThat(実際).isEqualTo(期待)`** と書け、メソッド名がそのまま意図になる。
- コレクション（`hasSize`・`contains`・`extracting`）、例外（`assertThatThrownBy`）、null 判定など、検証パターンごとに専用メソッドがあり、**1行で意図が伝わる**。
- IDE 補完が効くので、`assertThat(x).` と打てば使えるアサーションが一覧で出る。検証の書きやすさが段違い。

## 基本の書き方（コード）
```java
import static org.assertj.core.api.Assertions.*;

@Test
void 基本の検証() {
    String name = "Taro";

    // すべて assertThat(実際) から始まり、チェーンで条件を足す
    assertThat(name)
        .isNotNull()
        .isEqualTo("Taro")
        .startsWith("Ta")
        .hasSize(4);

    assertThat(1 + 2).isEqualTo(3);
    assertThat(0.1 + 0.2).isCloseTo(0.3, within(0.0001));   // 浮動小数
}
```
```java
// コレクションの検証
@Test
void リストの検証() {
    List<String> users = List.of("Taro", "Jiro", "Saburo");

    assertThat(users)
        .hasSize(3)
        .contains("Jiro")                    // 含む
        .containsExactly("Taro", "Jiro", "Saburo")   // 順序込みで完全一致
        .doesNotContain("Shiro");

    // extracting：要素の特定フィールドだけ取り出して検証
    List<Order> orders = orderService.findAll();
    assertThat(orders)
        .extracting(Order::getStatus)
        .containsOnly("PAID", "NEW");
}
```
```java
// 例外の検証（Mockito の verify と並ぶ定番）
@Test
void 在庫不足で例外() {
    assertThatThrownBy(() -> service.place(1L))
        .isInstanceOf(OutOfStockException.class)
        .hasMessageContaining("在庫");

    // 例外が出ないことの検証
    assertThatCode(() -> service.place(2L)).doesNotThrowAnyException();
}
```
```java
// オブジェクトのフィールド一括検証
@Test
void オブジェクトの検証() {
    Order order = service.create("NEW");

    assertThat(order)
        .extracting(Order::getId, Order::getStatus)
        .containsExactly(1L, "NEW");

    // 参照同一性ではなくフィールド単位で比較
    assertThat(order).usingRecursiveComparison()
        .isEqualTo(new Order(1L, "NEW"));
}
```

## 実務での使い方・定番パターン
- **`assertThat(実際)` で統一**：JUnit の `assertEquals` を混在させず、検証は AssertJ に寄せると引数順ミスが消え、読みやすさが揃う。
- **例外は `assertThatThrownBy`**：型・メッセージ・原因まで1チェーンで検証できる。JUnit の `assertThrows` でも可だが、メッセージ検証は AssertJ が簡潔。
- **`extracting` でフィールド検証**：DTO 変換・マッピングの確認で、要素の特定フィールドだけ取り出して `containsExactly` する。ネスト全体は `usingRecursiveComparison`。
- **コレクション系を活用**：`hasSize`・`containsExactlyInAnyOrder`・`allMatch` などで、ループを書かずに集合の性質を1行で表す。
- **Spring の検証でも頻出**：`@DataJpaTest` のクエリ結果や `@WebMvcTest` のレスポンス変換結果を `assertThat(...).hasSize(...)` で検証する（→ [testing.md](./testing.md)）。

## ハマりどころ / アンチパターン
- **`isEqualTo` は equals 依存**：`equals` を実装していないオブジェクト同士は参照比較で落ちる。フィールド比較したいなら **`usingRecursiveComparison()`** か `extracting` を使う。
- **`contains` と `containsExactly` の混同**：`contains` は「含む（順不同・部分一致）」、`containsExactly` は「順序込みで過不足なく一致」。意図と違うものを選ぶと誤検証になる。
- **import ミス**：`org.assertj.core.api.Assertions.assertThat` を static import する。Mockito にも `assertThat` 風のものがあり別 import を入れると衝突する。
- **チェーンの途中で代入を忘れる**：AssertJ は基本イミュータブルに条件を連結する。`assertThat(x)` を変数に取って後から `.isEqualTo` しても、同じオブジェクトに対する連鎖なので問題ないが、**1文で繋ぐ**のが読みやすい。
- **浮動小数を `isEqualTo` で比較**：誤差で落ちる。`isCloseTo(値, within(誤差))` を使う。
- **失敗時の `as` 説明を省く**：ループ内検証などで `as("ユーザ%dの状態", i)` を付けると、どのケースで落ちたか即わかる。

## 関連
[testing.md](./testing.md) / [junit5.md](./junit5.md) / [mockito.md](./mockito.md)
