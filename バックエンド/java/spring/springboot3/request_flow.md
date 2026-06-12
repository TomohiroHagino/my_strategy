# リクエストの流れ・各層は何を返すか（Spring Boot 3）

## ひとことで言うと
1リクエストが **Controller → Service → Repository → DB** と降り、**Entity が逆向きに上がってくる**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（POSTなら request body = JSON）
   ▼
[Controller]  @RequestBody でリクエストDTOを受け取る／Serviceを呼ぶ      ← Bean化・DI
   │
   ▼
[Service]     業務ロジック・トランザクション／Repositoryを呼ぶ          ← Bean化・DI
   │
   ▼
[Repository]  DBアクセス（JPA）                                        ← Bean化・DI（自動実装）
   │
   ▼
  DB ──→ Entity(モデル) を返す ─┐
   ▲                            │
[Repository] が Entity を Service に返す
   ▲
[Service]    が Entity（or 業務DTO）を Controller に返す
   ▲
[Controller] が レスポンスDTO に詰め替えて返す
   │ レスポンス（response body = JSON）
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す | Bean化？ |
|---|---|---|---|
| **Controller** | リクエストDTO（`@RequestBody`）/ パス変数 | **レスポンスDTO**（クライアントへ） | ✅ Bean（`@RestController`） |
| **Service** | （Controllerから）DTO/値 | **Entity or 業務DTO**（Controllerへ） | ✅ Bean（`@Service`） |
| **Repository** | （Serviceから）id / 条件 | **Entity**（DBから） | ✅ Bean（`@Repository`・自動実装） |
| **Entity / DTO** | — | データそのもの（運ばれる側） | ❌ Beanではない（毎回 new される値） |

- **Bean化されるのは「処理する部品」（Controller / Service / Repository）**。起動時にコンテナが1個ずつ作ってDI注入する。
- **Entity と DTO は「運ばれるデータ」**。Beanではなく、リクエストごとに生成される使い捨て（Jackson やリポジトリが生成）。

## コードで通して見る
```java
// 1) Controller：受け取り → Service呼び出し → レスポンスDTOを返す
@RestController
@RequestMapping("/users")
class UserController {
  private final UserService service;
  UserController(UserService service){ this.service = service; }   // DI（Bean化）

  @PostMapping
  UserResponse create(@RequestBody UserRequest req){     // req = リクエストDTO（毎回new・Beanではない）
    User user = service.register(req);                    // Service が Entity を返す
    return new UserResponse(user.getId(), user.getName()); // レスポンスDTOに詰め替えて返す
  }
}

// 2) Service：業務処理／Repositoryを呼び、Entityを返す
@Service
class UserService {
  private final UserRepository repo;
  UserService(UserRepository repo){ this.repo = repo; }   // DI（Bean化）

  @Transactional
  public User register(UserRequest req){
    User user = new User(req.name());   // DTO → Entity に変換
    return repo.save(user);             // Repository が Entity を返す
  }
}

// 3) Repository：DBアクセス。Entityを返す（実装はSpringが自動生成）
interface UserRepository extends JpaRepository<User, Long> {}
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：Repository→Service は Entity、Service→Controller は Entity か業務DTO、Controller→クライアントは必ず**レスポンスDTO**。
- **Entityをクライアントに直接返さない**：内部構造の露出・双方向関連の無限ループ・不要項目を避けるため、境界で DTO に詰め替える。→ [entity_or_dto.md](./entity_or_dto.md)
- **変換（mapper）の置き場**：手書き or MapStruct。Service か専用 mapper クラスで Entity↔DTO を変換。
- **トランザクション境界は Service**：`@Transactional` は Service に置くのが定番（Controllerには置かない）。→ [transactions.md](./transactions.md)

## ハマりどころ / アンチパターン
- **Controllerに業務やDBアクセスを書く**：層が崩れる。Controllerは「受けて・呼んで・詰め替えて返す」だけ。
- **Entityを `@RequestBody`/レスポンスに使い回す**：入力・出力・DBの都合が混ざる。リクエストDTO・レスポンスDTO・Entityは分ける。
- **Service/Repository を手動で `new` する**：`new UserService()` だとDIが効かず依存が解決されない。型ヒントでコンテナに任せる。→ [ioc_di.md](./ioc_di.md)
- **双方向関連のまま Entity を JSON 化**：無限ループ / N+1。DTOに詰め替える。

## 関連
[controller.md](./controller.md) / [service.md](./service.md) / [repository.md](./repository.md) / [entity_or_dto.md](./entity_or_dto.md) / [ioc_di.md](./ioc_di.md)
