# エンティティ / JPA / Hibernate / N+1（Spring Boot 3）

## ひとことで言うと
**JPA（Jakarta Persistence API）は「JavaオブジェクトとDBテーブルを対応づける仕様」**で、**Hibernate がその実装**（Spring Boot既定）。`@Entity` を付けたクラス＝テーブル、そのインスタンス＝1行、として扱う。

## 役割・なぜ必要か
- 生SQLを書かずに、Javaのオブジェクト操作でDBの読み書き・関連辿りを行うためにある（ORM）。
- **JPAは仕様（インターフェース群）、Hibernateは中身**という関係。アプリは原則 JPA 標準アノテーション（`jakarta.persistence.*`）に書き、実装はHibernateが担う。
- Spring Boot 3 では **`javax.persistence.*` → `jakarta.persistence.*`** に全面移行（2系からの最大の破壊的変更）。importを間違えると動かない。

---

## 基本の書き方（コード）
```java
import jakarta.persistence.*;          // Boot3は jakarta（javaxではない）
import java.time.LocalDateTime;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // DBの自動採番(AUTO_INCREMENT)
    private Long id;

    @Column(nullable = false, unique = true, length = 255)
    private String email;

    @Column(nullable = false)
    private String name;

    private LocalDateTime createdAt;

    protected User() {}                  // JPAは引数なしコンストラクタを要求
    // getter/setter は省略
}
```
- `@Id`＝主キー、`@GeneratedValue`＝採番戦略（`IDENTITY`/`SEQUENCE`/`AUTO`）。
- `@Column` で NOT NULL・unique・桁数などを指定。エンティティ定義とDB制約は**二重で**張るのが堅い。

## 実務での使い方・定番パターン

### 関連（アソシエーション）と fetch type
```java
@Entity
public class Post {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 多対一：Post 多 → User 1。@ManyToOne の既定 fetch は EAGER（要注意）
    @ManyToOne(fetch = FetchType.LAZY)   // ★明示的に LAZY 推奨
    @JoinColumn(name = "user_id")
    private User author;

    // 一対多：User 1 → Post 多。既定 fetch は LAZY
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
}
```
| 関連 | 既定 fetch |
|------|-----------|
| `@ManyToOne` / `@OneToOne` | **EAGER**（即時ロード） |
| `@OneToMany` / `@ManyToMany` | **LAZY**（遅延ロード） |

- **LAZY**：関連は実際にアクセスした時に初めてSQLを発行（プロキシ）。
- **EAGER**：親を取ると常に関連もJOIN/追加クエリで取る。**乱用するとN+1や不要JOINの温床**。基本は **全部 LAZY 明示**にして、必要な時だけ fetch join で取る方針が安全。

### 多対多
```java
@ManyToMany
@JoinTable(name = "post_tags",
    joinColumns = @JoinColumn(name = "post_id"),
    inverseJoinColumns = @JoinColumn(name = "tag_id"))
private Set<Tag> tags = new HashSet<>();
```
- 実務では中間テーブルに属性が増えがちなので、**中間エンティティ＋2つの`@ManyToOne`**に分解することが多い。

## N+1問題とは（最重要の罠）
一覧で親をN件取り、ループで LAZY 関連を1件ずつ辿ると、**最初の1回＋N回**のSQLが走る性能劣化。
```java
// NG：N+1。posts 1回 + 各 post.getAuthor() ごとに +1 で合計 1+N 回
List<Post> posts = postRepository.findAll();
for (Post p : posts) {
    System.out.println(p.getAuthor().getName());  // LAZYが毎回SQL発行
}
```

### 解消1：fetch join（JPQLで一発で取る）
```java
@Query("SELECT p FROM Post p JOIN FETCH p.author")
List<Post> findAllWithAuthor();   // JOINで author を同時取得（追加クエリなし）
```

### 解消2：@EntityGraph（メソッドに付けるだけ）
```java
@EntityGraph(attributePaths = {"author", "comments"})
@Query("SELECT p FROM Post p")
List<Post> findAllWithGraph();
// 派生クエリにも付けられる
@EntityGraph(attributePaths = "author")
List<Post> findByAuthorActiveTrue();
```
- コレクション(`@OneToMany`)を複数 fetch join するとデカルト積になる。`Set`化や `distinct`、必要なら **`@BatchSize` / `default_batch_fetch_size`** で「IN句まとめ取り」に切り替える。

## ハマりどころ / アンチパターン
- **N+1（LAZY＋ループ）**（最頻）→ fetch join か `@EntityGraph` で解消。検出は `spring.jpa.show-sql=true` やp6spy・Hibernate統計でSQL回数を見る。
- **EAGER乱用**：常に関連を引いてしまい、不要なJOIN/クエリで重くなる。基本 LAZY 明示。
- **双方向関連の `toString` / JSON で無限ループ**：親→子→親…で `StackOverflowError`。`toString`/`equals` に相手側を含めない、JSONは **EntityをそのままレスポンスにせずDTOへ**（→ [dto.md](./dto.md)）。Jacksonなら `@JsonManagedReference`/`@JsonBackReference` もあるが、DTO化が王道。
- **`equals`/`hashCode` の自動生成**：全フィールドや関連を含めると遅延ロード誘発・無限ループ。**ビジネスキー（例: email）だけ**で組むか、生成IDのみで慎重に。
- **`LazyInitializationException`**：トランザクション/セッションが閉じた後に LAZY 関連へアクセスすると発生。**取得時に fetch しておく**か、サービス層（トランザクション内）でDTOに詰め替える。
- **import間違い**：`javax.persistence.*` のままだとBoot3で動かない。必ず `jakarta.persistence.*`。

## 関連: [repository.md](./repository.md) / [dto.md](./dto.md)
