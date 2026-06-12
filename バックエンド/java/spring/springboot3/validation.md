# バリデーション（Bean Validation）（Spring Boot 3）

## ひとことで言うと
リクエストで受け取った値（DTO のフィールドなど）が**正しい形か**を、`@NotNull` `@Size` `@Email` などの**アノテーションで宣言的に検証する仕組み**。Boot3 では **Jakarta Bean Validation**（`jakarta.validation.*`）が標準。

## 役割・なぜ必要か
- 「名前は必須」「メールは形式が正しい」「年齢は0以上」といった入力ルールを、**コントローラやサービスのコードに if 文を散らさず**、DTO のフィールドに宣言として書ける。
- 検証ロジックを DTO（型）に集約することで、入口（Controller）で一括チェックでき、業務ロジックは「値はもう正しい」前提で書ける。Rails の Strong Parameters が「許可リスト」だったのに対し、こちらは**値の正当性検証**を担う（許可と検証は別物）。
- Boot2 → Boot3 の移行で `javax.validation.*` → **`jakarta.validation.*`** にパッケージが全面変更。import を間違えるとアノテーションが効かない。
- 依存は `spring-boot-starter-validation` を明示的に追加する必要がある（web starter に含まれないので忘れがち）。

## 基本の書き方（コード）
```java
// build.gradle に implementation 'org.springframework.boot:spring-boot-starter-validation'

// DTO（リクエストボディ）
import jakarta.validation.constraints.*;

public record UserCreateRequest(
    @NotBlank(message = "名前は必須です")
    @Size(max = 50, message = "名前は50文字以内")
    String name,

    @NotBlank @Email(message = "メール形式が不正です")
    String email,

    @NotNull @Min(value = 0, message = "年齢は0以上")
    Integer age
) {}
```

```java
// コントローラ引数に @Valid を付けると、ボディが検証される
import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public UserResponse create(@Valid @RequestBody UserCreateRequest req) {
        // ここに来た時点で req は検証済み。違反があれば
        // MethodArgumentNotValidException が投げられ、メソッドは呼ばれない
        return userService.create(req);
    }
}
```

```java
// 主なアノテーション（jakarta.validation.constraints）
// @NotNull   … null を禁止（空文字 "" は通る）
// @NotBlank  … 文字列の null・空・空白のみを禁止（String 専用）
// @NotEmpty  … null・空コレクション/空文字を禁止（length>0 を要求）
// @Size(min=, max=)  … 文字列長・コレクション要素数
// @Email     … メール形式
// @Min/@Max  … 数値の下限・上限
// @Pattern(regexp=) … 正規表現
// @Past/@Future     … 日時の過去/未来
```

## 実務での使い方・定番パターン
- **`@Valid` 付け忘れに注意**：アノテーションを書いても `@Valid`（または `@Validated`）が無いと**一切検証されない**。最頻出のハマり。
- **ネストした DTO** は、親フィールドに `@Valid` を付けて初めて子も検証される。
  ```java
  public record OrderRequest(
      @NotNull @Valid AddressDto address,        // @Valid が無いと中身は検証されない
      @NotEmpty List<@Valid OrderLine> lines     // 要素ごとの検証は List<@Valid ...>
  ) {}
  ```
- **`@Validated`（Spring 製）とグループ**：クラスに `@Validated` を付け、**作成時と更新時でルールを変える**などグループ分けができる。
  ```java
  public interface OnCreate {}
  public interface OnUpdate {}

  public record UserRequest(
      @Null(groups = OnCreate.class)            // 作成時は id を送ってはダメ
      @NotNull(groups = OnUpdate.class)         // 更新時は id 必須
      Long id,
      @NotBlank String name
  ) {}

  // コントローラ側でグループを指定
  @PostMapping
  public void create(@Validated(OnCreate.class) @RequestBody UserRequest req) { ... }
  ```
- **パスやクエリの単項目検証**：クラスに `@Validated` を付け、引数に直接制約を書く。違反時は `ConstraintViolationException`。
  ```java
  @RestController
  @Validated   // メソッド引数の制約を有効化するために必須
  public class ItemController {
      @GetMapping("/items/{id}")
      public Item get(@PathVariable @Min(1) Long id) { ... }
  }
  ```
- **カスタムバリデータ**：標準アノテーションで表せないルール（例：ユーザー名の重複なし）は自作する。
  ```java
  @Target(ElementType.FIELD)
  @Retention(RetentionPolicy.RUNTIME)
  @Constraint(validatedBy = UniqueEmailValidator.class)
  public @interface UniqueEmail {
      String message() default "既に使われているメールです";
      Class<?>[] groups() default {};
      Class<? extends Payload>[] payload() default {};
  }

  public class UniqueEmailValidator
      implements ConstraintValidator<UniqueEmail, String> {
      private final UserRepository repo;
      public UniqueEmailValidator(UserRepository repo) { this.repo = repo; } // DI 可能
      @Override
      public boolean isValid(String email, ConstraintValidatorContext ctx) {
          if (email == null) return true;          // null は @NotNull に任せる
          return !repo.existsByEmail(email);
      }
  }
  ```

## ハマりどころ / アンチパターン
- **`@Valid` / `@Validated` 付け忘れ**：アノテーションだけ書いて引数に付け忘れ、検証が走らない。「不正値が素通りする」の筆頭原因。
- **`javax.validation.*` を import**：Boot3 では効かない。**`jakarta.validation.*`** に統一する。IDE が古い候補を出すので要確認。
- **starter 追加忘れ**：`spring-boot-starter-validation` が無いと `@Email` 等が無視される。
- **`@NotNull` と `@NotBlank` の混同**：文字列の「空文字禁止」には `@NotBlank`。`@NotNull` は `""` を通してしまう。
- **エラーレスポンスをコントローラで整形**：try/catch を各メソッドに書くのはアンチパターン。**例外ハンドラに集約**する（`MethodArgumentNotValidException` を `@RestControllerAdvice` で捕捉して整形）。→ [exception_handling.md](./exception_handling.md)
- **検証は許可リストではない**：Strong Parameters のような「受け取るキーの制限」ではない。DTO に無いフィールドは Jackson が無視するだけ。受け取る形は DTO の設計で制御する。→ [dto.md](./dto.md)

## 関連: [controller.md](./controller.md) / [exception_handling.md](./exception_handling.md) / [dto.md](./dto.md)
