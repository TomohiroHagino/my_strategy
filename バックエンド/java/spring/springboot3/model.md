# モデル（Model）（Spring Boot 3）

## ひとことで言うと
Springには **「Model」という単一の層は無い**。Rails の Fat Model（データ＋業務ロジックを1クラスに）に当たるものが、Springでは**役割ごとに分割**されている。このファイルはその“地図”。

> 「modelが無い」のではなく、**3つ＋αに分かれている**だけ。下の対応を押さえれば迷わない。

## Springでの「モデル」の正体（対応表）
| Railsの Fat Model がやっていたこと | Springでの担当 | 詳細 |
|---|---|---|
| テーブルに対応するデータ構造 | **Entity**（`@Entity`） | [entity_jpa.md](./entity_jpa.md) |
| DBアクセス（検索・保存） | **Repository**（`JpaRepository`） | [repository.md](./repository.md) |
| 業務ロジック・トランザクション | **Service**（`@Service`） | [service.md](./service.md) |
| 外部に見せる形 | **DTO**（`record` 等） | [dto.md](./dto.md) |

## 役割・なぜ分けるのか
- **単一責任**。Railsの Fat Model は「データ＋ロジック＋永続化」を1か所に集めて肥大しがち。Springは最初から層で分離し、テスト・再利用・置き換えをしやすくする。
- 「データの器（Entity）」と「業務の手順（Service）」と「DB操作（Repository）」を混ぜない、というのがSpring流の設計思想。

## 基本の書き方（3分割のイメージ）
```java
// ① Entity = データ構造（テーブル↔クラス）
@Entity
class User {
    @Id @GeneratedValue Long id;
    String email;
    // ロジックは基本ここに詰めない（貧血でよい、が一般的）
}

// ② Repository = DBアクセス（インターフェースだけでCRUDが生える）
interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

// ③ Service = 業務ロジック（Repositoryを使う）
@Service
class UserService {
    private final UserRepository repo;
    UserService(UserRepository repo) { this.repo = repo; }

    @Transactional
    public User register(String email) {
        return repo.save(new User(email));   // ロジックはここ
    }
}
```

## 用語の罠（もう一つの「Model」）
**MVC文脈の「Model」**は別物で、**Controller が View（Thymeleaf）に渡すデータの入れ物**（`Model` / `ModelAndView`）を指すこともある。
```java
@GetMapping("/users")
String list(Model model) {            // ← これもSpringでは "Model"
    model.addAttribute("users", service.findAll());
    return "users";                   // Thymeleafテンプレート名
}
```
「JPAのエンティティ」と「MVCのModelオブジェクト」は**別物**。文脈で読み分ける。
→ この「Modelが3つの意味で出てくる」混乱と `ModelAndView` の詳しい図解は **[view.md の「Model と View の関係」](./view.md#最重要model-と-view-の関係一番つまずく所)** に保存（一番つまずく所）。

## ハマりどころ / アンチパターン
- **Entityに業務ロジックを詰めすぎる**：Springでは基本Serviceに置く（Railsの感覚で書くとズレる）。
- **Entityを直接外に返す**：内部構造の漏れ・遅延ロード例外。DTOへ。→ [dto.md](./dto.md)
- **Repositoryにロジックを書く**：Repositoryは永続化に専念、判断はService。

## 関連
[entity_jpa.md](./entity_jpa.md) / [repository.md](./repository.md) / [service.md](./service.md) / [dto.md](./dto.md) / [view.md](./view.md)
