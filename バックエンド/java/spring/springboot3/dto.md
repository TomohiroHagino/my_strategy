# DTO（Entityと分ける理由）（Spring Boot 3）

## ひとことで言うと
**DTO（Data Transfer Object）**は、層やシステム境界（API↔クライアント）でデータを運ぶための**入出力専用オブジェクト**。DBに紐づく Entity とは別物として用意し、リクエスト/レスポンスの形を表す。

## 役割・なぜ必要か
- **API契約とDB構造を分離する**：エンティティ（テーブル定義）が変わってもAPIの形を保てる／逆にAPIの都合でDB構造を歪めない。
- **内部の過剰公開を防ぐ**：`passwordHash`・内部フラグ・管理用カラムなどを、Entityをそのまま返すと**漏らしてしまう**。DTOなら出す項目を明示できる。
- **`LazyInitializationException` を回避する**：Entityをコントローラまで持ち出してJSON化すると、トランザクション外でLAZY関連に触れて例外になる。**トランザクション内でDTOへ詰め替えれば安全**（→ [entity_jpa.md](./entity_jpa.md)）。

---

## 基本の書き方（コード）
Boot3（Java 17+）では **`record`** がDTOに最適（不変・ボイラープレート無し）。
```java
// レスポンス用DTO（出力する項目だけを宣言）
public record UserResponse(
    Long id,
    String name,
    String email
) {
    // EntityからDTOへ変換するファクトリ（トランザクション内で呼ぶ）
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail());
        // passwordHash や内部フラグは「載せない」＝漏れない
    }
}
```
```java
// リクエスト用DTO（受け取る項目だけ。検証アノテーションも付く）
import jakarta.validation.constraints.*;

public record CreateUserRequest(
    @NotBlank String name,
    @Email @NotBlank String email,
    @Size(min = 8) String password
) {}
```
- **入力用と出力用は分ける**：受け取りたい項目（password）と返したい項目（id）は一致しないため。

## 実務での使い方・定番パターン

### Controller では Entity を一切外に出さない
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;
    public UserController(UserService userService) { this.userService = userService; }

    @PostMapping
    public UserResponse create(@RequestBody @Valid CreateUserRequest req) {
        User saved = userService.register(req);   // Service内（Tx内）で永続化
        return UserResponse.from(saved);          // ★EntityでなくDTOを返す
    }

    @GetMapping("/{id}")
    public UserResponse get(@PathVariable Long id) {
        return userService.findById(id);          // Serviceが UserResponse を返す設計も可
    }
}
```

### マッピング：手動 vs MapStruct
**手動マッピング**（ファクトリメソッド / 小規模・明示的）：
```java
return new UserResponse(u.getId(), u.getName(), u.getEmail());
```
**MapStruct**（コンパイル時にマッパー自動生成・大規模向け）：
```java
import org.mapstruct.Mapper;

@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);                 // 同名フィールドを自動対応
    List<UserResponse> toResponseList(List<User> users);
}
```
- 小〜中規模は**手動 record + ファクトリで十分**（KISS）。フィールドが多くマッピングが反復的になったら MapStruct を検討（DRY）。リフレクション系（ModelMapper）は型安全性・性能で劣りがち。

### ネスト/一覧は最初からDTOに射影する
N+1や余計なロードを避けるため、JPQLで**DTOへ直接射影**（コンストラクタ式）するのも定番。
```java
@Query("SELECT new com.example.dto.UserSummary(u.id, u.name) FROM User u")
List<UserSummary> findAllSummaries();   // Entityを経由せず必要列だけ取得
```

## ハマりどころ / アンチパターン
- **Entityを直接JSONで返す**（最頻の地雷）：
  - 内部項目（パスワードハッシュ・権限フラグ等）が**そのまま漏れる**。
  - LAZY関連にレスポンス生成時に触れて **`LazyInitializationException`**。
  - 双方向関連で**無限ループ**（→ DTO化で断ち切る）。
  - DB列名の変更がそのままAPI破壊につながる。→ **必ずDTOへ詰め替える**。
- **入力DTOと出力DTOを共用**：受け取りたくない項目（id）を受けたり、返したくない項目を露出する原因。用途別に分ける。
- **過剰マッピング**：使いもしない全項目を律儀にマッピングしてDTOが肥大化。**そのユースケースで必要な項目だけ**にする（YAGNI）。
- **Entity→DTO変換をトランザクション外でやる**：LAZY関連を踏んで例外。変換はサービス層（Tx内）で完了させる。
- **DTOにロジックを盛る**：DTOは原則データの入れ物。業務ロジックはService側へ。

## 関連: [controller.md](./controller.md) / [entity_jpa.md](./entity_jpa.md)
