# レイヤード構成（Spring Boot 3）

## ひとことで言うと
Springアプリの基本形は **Controller → Service → Repository → Entity の3層（＋Entity）**。「受付・業務ロジック・DBアクセス・データ」を層で分け、各層は **DIコンテナが生成して結びつける**。上の層が下の層を呼ぶ一方向の依存にする。

## 役割・なぜ必要か
- **責務分離**：1つのクラスに「HTTPの解釈」「業務ルール」「SQL」を混ぜると、変更が波及し、テストもできない。層で切れば各々を独立に差し替え・テストできる。
- 各層の担当：
  - **Controller**…URL↔メソッドの対応、入力（JSON/クエリ）→オブジェクト変換、出力の組み立て。HTTPの都合はここだけ。
  - **Service**…業務ロジック、トランザクション境界（`@Transactional`）、複数Repositoryの調整。アプリの本体。
  - **Repository**…DBアクセス。Spring Data JPAなら `JpaRepository` を継承するだけでCRUDが揃う。
  - **Entity**…テーブルに対応するデータの入れ物（`@Entity`）。
- **DTOで層の境界を切る**：外部に見せる形（API）と内部のEntity（DB構造）を分離し、片方の変更がもう片方を壊さないようにする。→ [dto.md](./dto.md)

## 基本の書き方（コード）
パッケージは「層ごと（package by layer）」が入門の定番。
```
com.example.demo
├── DemoApplication.java          // mainはルートに置く（scan基点）
├── controller/  UserController.java
├── service/     UserService.java
├── repository/  UserRepository.java
├── entity/      User.java
└── dto/         UserResponse.java / CreateUserRequest.java
```
Entity（データの入れ物。Boot 3 は `jakarta.persistence`）。
```java
// entity/User.java
package com.example.demo.entity;
import jakarta.persistence.*;

@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    // getter/setter 省略
}
```
Repository（継承だけでCRUD）。
```java
// repository/UserRepository.java
package com.example.demo.repository;
import com.example.demo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, Long> {
    boolean existsByEmail(String email); // メソッド名から自動でクエリ生成
}
```
Service（業務ロジック・トランザクション。コンストラクタ注入）。
```java
// service/UserService.java
package com.example.demo.service;
import com.example.demo.repository.UserRepository;
import com.example.demo.entity.User;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; } // DI

    @Transactional
    public User register(String name, String email) {
        if (repo.existsByEmail(email)) throw new IllegalStateException("email taken");
        User u = new User();
        u.setName(name); u.setEmail(email);
        return repo.save(u);
    }
}
```
Controller（HTTPの入口。EntityではなくDTOを返す）。
```java
// controller/UserController.java
package com.example.demo.controller;
import com.example.demo.service.UserService;
import com.example.demo.dto.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService service;
    public UserController(UserService service) { this.service = service; }

    @PostMapping
    public UserResponse create(@RequestBody CreateUserRequest req) {
        var u = service.register(req.name(), req.email());
        return new UserResponse(u.getId(), u.getName()); // Entity→DTOに詰め替え
    }
}
```

## 実務での使い方・定番パターン
- **package by layer vs package by feature**：小〜中規模は layer（上記）で十分。大規模・複数チームなら **feature単位**（`user/`, `order/` の中に controller/service/repository を入れる）にして、機能ごとに凝集させる。
- **層は飛ばさない**：ControllerからRepositoryを直接呼ばず、必ずServiceを通す。トランザクションと業務ルールの置き場を一本化できる。
- **DTOは入口（Request）と出口（Response）を分ける**：required項目や見せたくないフィールド（パスワード等）を型で区別できる。→ [dto.md](./dto.md)
- **インターフェース＋実装**は必要になってから。`UserService` を即interface化するのはYAGNI。差し替え（複数実装/モック）が現実に要るときに切る。
- **例外は層をまたいでControllerAdviceに集約**し、各層ではドメイン例外を投げるだけにする。→ [exception_handling.md](./exception_handling.md)

## ハマりどころ / アンチパターン
- **Entityをそのままcontrollerで返す**：DBの内部構造がAPIに漏れ、lazy loading中にJSON化して `LazyInitializationException`、循環参照で無限ループ、機密フィールド露出…の温床。必ずDTOに詰め替える。→ [entity_jpa.md](./entity_jpa.md)
- **Fat Controller**：Controllerに業務ロジックを書くと、テストにMockMvcやHTTPが必要になり、再利用も効かない。ロジックはServiceへ。
- **Fat Service / God Service**：1つのServiceが何百行・全機能を抱える。feature単位や小さなServiceに割って凝集度を上げる。
- **層をまたぐ循環依存**：ServiceA→ServiceB→ServiceA のような相互参照は、起動時に `BeanCurrentlyInCreationException` を招く。共通処理を別Beanへ抽出して依存を一方向に整理する。→ [ioc_di.md](./ioc_di.md)
- **RepositoryにビジネスロジックやEntityにHTTPの都合**を持ち込む（責務違反）。各層は自分の担当だけにする。
- **DTOとEntityの詰め替えを毎回手書き**してドリフト。規模が出たらMapStruct等のマッパーで一元化する（過剰になる前に導入）。

## 関連
[controller.md](./controller.md) / [service.md](./service.md) / [repository.md](./repository.md) / [dto.md](./dto.md) / [ioc_di.md](./ioc_di.md)
