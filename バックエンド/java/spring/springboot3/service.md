# サービス層（@Service）（Spring Boot 3）

## ひとことで言うと
**業務ロジック（ビジネスルール）とトランザクション境界を担う層**。Controller から呼ばれ、Repository を使って DB を操作する「アプリの本体」。`@Service` を付けてBean化する。

## 役割・なぜ必要か
- 「何をどう処理するか」という**判断・計算・複数操作の組み合わせ**をここに集める。Controller（入出力変換）・Repository（DBアクセス）と責務を分けることで、テスト・再利用・変更が楽になる。
- **薄いController-厚いService**が原則。Controller は受付に徹し、ロジックは Service に寄せる。→ [controller.md](./controller.md)
- **トランザクション境界は基本ここ**。「ひとまとまりの業務処理（複数の保存・更新）」を1つの `@Transactional` でくくり、途中失敗なら全部ロールバックして整合性を守る。→ [transactions.md](./transactions.md)

## 基本の書き方（コード）
```java
@Service
public class UserService {

    private final UserRepository userRepository;     // Repositoryをコンストラクタ注入
    private final MailSender mailSender;

    public UserService(UserRepository userRepository, MailSender mailSender) {
        this.userRepository = userRepository;
        this.mailSender = mailSender;
    }

    // 参照だけなら readOnly=true（最適化のヒント）
    @Transactional(readOnly = true)
    public UserResponse findById(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new UserNotFoundException(id));
        return UserResponse.from(user);   // EntityはDTOに詰め替えて返す
    }

    // 書き込みは @Transactional（既定で実行時例外時にロールバック）
    @Transactional
    public UserResponse create(CreateUserRequest req) {
        if (userRepository.existsByEmail(req.email())) {
            throw new DuplicateEmailException(req.email());   // 業務ルールで弾く
        }
        User saved = userRepository.save(User.create(req.name(), req.email()));
        mailSender.sendWelcome(saved.getEmail());
        return UserResponse.from(saved);
    }
}
```

## 実務での使い方・定番パターン
- **コンストラクタ注入を使う**：`@Autowired` フィールド注入より、`final` で不変・テストでモック差し替えが容易・依存が明示される。引数が1つでも Spring が自動で注入する（注釈不要）。
- **`@Transactional` は Service メソッドに置く**：1メソッド=1業務単位=1トランザクションが基本。複数リポジトリ操作を**まとめて**くくり、途中で例外なら全ロールバック。
- **読み取り専用は `readOnly = true`**：参照系メソッドに付けると、Hibernate のダーティチェック省略などで軽くなる。
- **業務例外を投げ、整形は外で**：`UserNotFoundException` 等の意味のある例外を投げ、HTTPステータスへの変換は `@ControllerAdvice` に任せる（Service は HTTP を知らない）。→ [exception_handling.md](./exception_handling.md)
- **Entityの詰め替えは Service の出口で**：Controller には DTO を返す。Service 内で `UserResponse.from(entity)` のように変換する。→ [dto.md](./dto.md)
- **ロールバック対象の指定**：既定では `RuntimeException` / `Error` でロールバック。**チェック例外でもロールバックしたい**なら `@Transactional(rollbackFor = Exception.class)` を明示する。

## ハマりどころ / アンチパターン
- **自己呼び出し（同一クラス内メソッド呼び出し）で `@Transactional` が効かない（最頻出の罠）**：`@Transactional` は **AOPプロキシ**で実現される。**同じクラスの別メソッドを `this.xxx()` で呼ぶとプロキシを経由しない**ため、その内側の `@Transactional` / `@Async` が**無視される**。
```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order o) {
        validate(o);
        save(o);   // ← this.save() はプロキシを通らない！
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void save(Order o) {   // この REQUIRES_NEW は内部呼び出しでは効かない
        orderRepository.save(o);
    }
}
// 対処: トランザクション境界が要るメソッドは別Beanに切り出して注入し、
//       そのBean経由で呼ぶ（=プロキシを通す）。または自己注入/AspectJモード。
```
- **fat service（神サービス）**：1クラスに何百行・無関係なメソッドが集まると保守不能に。**ユースケース/ドメイン単位で分割**し、共通処理は別Beanへ抽出する（高凝集・低結合）。
- **トランザクション境界が広すぎ / 狭すぎ**：広すぎると DB ロックを長く握り性能劣化、狭すぎると複数操作が別トランザクションになり**途中失敗で不整合**。`@Transactional` は「1業務=1メソッド」に合わせて置く。Controller や Repository に置くのは避ける。
- **`@Transactional` 内で外部API/メール送信をして失敗時に整合が崩れる**：DBはロールバックされても**送信済みメールは戻らない**。コミット後に副作用を出したいなら `TransactionSynchronization`（afterCommit）等を使う。
- **例外を握りつぶしてロールバックされない**：`@Transactional` メソッド内で例外を `try-catch` して握りつぶすと、Springは例外を観測できず**コミットされてしまう**。ロールバックさせたい異常は再スローする。

## 関連
[controller.md](./controller.md) / [repository.md](./repository.md) / [transactions.md](./transactions.md)
