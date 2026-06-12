# リポジトリ（Spring Data JPA）（Spring Boot 3）

## ひとことで言うと
**Spring Data JPA のリポジトリ**は、`JpaRepository<Entity, ID>` を継承した**インターフェースを書くだけ**で、CRUDメソッドの実装をSpringが自動生成してくれる仕組み。自分で `EntityManager` を触る実装クラスを書かなくてよい。

## 役割・なぜ必要か
- DBアクセス（SQL/JPQL発行）を「Javaのメソッド呼び出し」に翻訳し、ボイラープレートな実装コードを消すためにある。
- レイヤード構成では **Service が Repository を呼び、Repository が DB を叩く**。Service は「何をするか」、Repository は「どう取り出すか」に責務を分ける。
- メソッド名の**命名規約**からクエリを推論する（派生クエリメソッド）。型安全で、リファクタ時もコンパイラが効く。

---

## 基本の書き方（コード）
```java
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.Optional;

// User=エンティティ, Long=主キー(@Id)の型
public interface UserRepository extends JpaRepository<User, Long> {

    // 派生クエリメソッド：メソッド名からSQLが自動生成される
    Optional<User> findByEmail(String email);     // WHERE email = ?
    List<User> findByActiveTrue();                // WHERE active = true
    List<User> findByNameContaining(String kw);   // WHERE name LIKE %kw%
    long countByActiveTrue();
    boolean existsByEmail(String email);
}
```
```java
// 継承するだけで使えるCRUD（自分で書かない）
userRepository.save(user);                 // INSERT or UPDATE（戻り値=保存後Entity）
userRepository.findById(1L);               // Optional<User> を返す
userRepository.findAll();                  // 全件（本番では基本使わない→Pageable）
userRepository.deleteById(1L);
userRepository.existsById(1L);
```

### 派生クエリメソッドの命名ルール（要点）
- `find` / `read` / `get` …取得、`count` …件数、`exists` …存在、`delete` …削除。
- 続けて `By` + プロパティ名（**エンティティのフィールド名**であってカラム名ではない）。
- 連結は `And` / `Or`、比較は `GreaterThan` / `LessThan` / `Between` / `Like` / `In` / `IsNull` / `True` / `False`、並びは `OrderBy...Asc/Desc`。
```java
List<User> findByAgeGreaterThanAndActiveTrueOrderByCreatedAtDesc(int age);
```

## 実務での使い方・定番パターン

### Optional<T> で「無いかもしれない」を表現する
`findById` / `findByXxx`（単一）は **`Optional<T>`** を返す。`null` ではなく Optional で受け、無い場合の分岐を明示する。
```java
User user = userRepository.findByEmail(email)
    .orElseThrow(() -> new UserNotFoundException(email));   // 無ければ例外
// もしくは .orElse(defaultUser) / .map(...).orElseGet(...)
```

### @Query：命名規約で表現しきれないクエリを書く
```java
public interface UserRepository extends JpaRepository<User, Long> {

    // JPQL（エンティティ名・フィールド名で書く。テーブル名ではない）
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
    Optional<User> findActiveByEmail(@Param("email") String email);

    // ネイティブSQL（DB固有関数を使いたい等。テーブル名・カラム名で書く）
    @Query(value = "SELECT * FROM users WHERE LOWER(email) = LOWER(:email)",
           nativeQuery = true)
    Optional<User> findByEmailIgnoreCaseNative(@Param("email") String email);

    // 更新系は @Modifying を付ける（戻り値=影響行数）。要トランザクション
    @org.springframework.data.jpa.repository.Modifying
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :limit")
    int deactivateStaleUsers(@Param("limit") java.time.LocalDateTime limit);
}
```

### Pageable / Sort：ページングと並び替え
全件取得は危険。一覧は必ず**ページング**する（無界クエリを避ける）。
```java
import org.springframework.data.domain.*;

// Controller/Service側
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());
Page<User> page = userRepository.findByActiveTrue(pageable);

page.getContent();        // List<User>（このページ分）
page.getTotalElements();  // 総件数
page.getTotalPages();
```
```java
// Repository側：引数に Pageable / Sort を足すだけ
Page<User> findByActiveTrue(Pageable pageable);
List<User> findByActiveTrue(Sort sort);
```
- `Page` は総件数のための `COUNT` クエリも走る。件数が要らないなら `Slice<T>`（次ページ有無だけ）が軽い。

### save の挙動
`save(entity)` は **新規なら INSERT、IDがあり既存なら UPDATE**（=upsertではなく、永続化コンテキスト基準で判定）。戻り値の Entity を使うこと（採番されたIDが入る）。

## ハマりどころ / アンチパターン
- **派生メソッドの命名ミス**（最頻）：`findByEmailAdress`（typo）や、存在しないフィールド名を書くと **起動時に `PropertyReferenceException` で落ちる**。逆に言えば実行前に気づける。フィールド名はエンティティ準拠（DBカラム名ではない）。
- **N+1問題**：一覧取得後にループで関連を辿ると、関連ごとにSQLが追加発行される。Repository単体でなく**フェッチ戦略で解く**（fetch join / `@EntityGraph`）。詳細は [entity_jpa.md](./entity_jpa.md)。
- **`Optional.get()` の乱用**：中身があるか確認せず `get()` すると `NoSuchElementException`。`orElseThrow` / `orElse` / `map` で安全に開く。
- **`findAll()` を本番一覧で使う**：無界クエリ。必ず `Pageable` を渡す。
- **`@Modifying` 付け忘れ／トランザクション無し**：UPDATE/DELETE の `@Query` は `@Modifying` 必須、かつ書き込みは `@Transactional`（多くはService側）。実行後、永続化コンテキストと食い違うため `clearAutomatically = true` が要る場合がある。
- **派生メソッドで複雑化しすぎる**：名前が `findByAAndBOrCAndDOrderByE...` のように長大化したら `@Query` か Specification / QueryDSL に切り替える。

## 関連: [entity_jpa.md](./entity_jpa.md) / [service.md](./service.md)
