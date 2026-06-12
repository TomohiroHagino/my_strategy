# コントローラ（@RestController）（Spring Boot 3）

## ひとことで言うと
HTTPリクエストを受け取り、**URL↔メソッドの対応付け・入力の取り出し・Serviceの呼び出し・レスポンス（主にJSON）の組み立て** を担う受付層。REST API では `@RestController` を使う。

## 役割・なぜ必要か
- ルーティング（`@GetMapping` 等）でメソッドに振り分け、`@PathVariable` / `@RequestParam` / `@RequestBody` で入力を取り出し、`@Service` を呼んで結果を返す「交通整理」の層。
- ここは **薄く**保つのが原則。判断・計算・トランザクションは Service へ寄せる（薄いController-厚いService）。→ [service.md](./service.md)
- `@RestController` = **`@Controller` + `@ResponseBody`**。`@Controller` は HTML（View）を返す MVC 用、`@RestController` は**戻り値をそのままレスポンスボディ（JSON）に変換**する API 用。この違いが分からず混同すると事故る。

## 基本の書き方（コード）
```java
@RestController
@RequestMapping("/api/users")   // クラス共通プレフィックス
public class UserController {

    private final UserService userService;   // Serviceをコンストラクタ注入

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users/123  → URLの一部を受け取る
    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id);   // 戻り値オブジェクトが自動でJSON化される
    }

    // GET /api/users?keyword=foo&page=0  → クエリパラメータ
    @GetMapping
    public List<UserResponse> search(
            @RequestParam(defaultValue = "") String keyword,
            @RequestParam(defaultValue = "0") int page) {
        return userService.search(keyword, page);
    }

    // POST /api/users  → JSONボディを受け取り、ステータスを制御
    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req) {
        UserResponse created = userService.create(req);
        // 201 Created + Location ヘッダを明示
        return ResponseEntity
                .created(URI.create("/api/users/" + created.id()))
                .body(created);
    }
}
```

## 実務での使い方・定番パターン
- **マッピング注釈**：`@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` は `@RequestMapping(method=...)` の短縮形。クラスに `@RequestMapping("/api/users")` を付けて共通化する。
- **入力の3種を使い分ける**：
  - `@PathVariable` … `/users/{id}` のようにリソース識別子をパスに埋める（RESTの定番）。
  - `@RequestParam` … `?keyword=...&page=...` の検索・絞り込み・ページング。`required`/`defaultValue` を活用。
  - `@RequestBody` … JSONボディを **DTOにデシリアライズ**（Jackson）。作成・更新で使う。
- **`ResponseEntity` でステータス・ヘッダを制御**：作成は 201、削除は 204、明示が要らない単純取得は戻り値直返し（200）でも良い。
```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> delete(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();   // 204 No Content
}
```
- **入力検証は `@Valid`**：`@RequestBody` と組み合わせて境界で弾く。失敗時の整形は `@ControllerAdvice` に寄せる。→ [validation.md](./validation.md) / [exception_handling.md](./exception_handling.md)
- **JSON変換は Jackson が自動**：`spring-boot-starter-web` で `ObjectMapper` が auto-configuration される。日付は `application.yml` の `spring.jackson.*` でフォーマット統一。
- **Entityを直接返さない**：レスポンスは必ず **DTO（response record）** を返す。→ [dto.md](./dto.md)

## ハマりどころ / アンチパターン
- **`@RestController` で View（テンプレート名）を返そうとする**：`@RestController` は戻り値文字列を**そのままレスポンスボディ**にするため、`"user/list"` を返すと HTML ではなく**文字列 "user/list" がそのまま**返る。HTML を返すなら `@Controller`、API なら DTO を返す、と用途で分ける。
- **`@RequestBody` のデシリアライズ失敗**：JSONのフィールド名/型が DTO と合わない、ボディが空、`Content-Type: application/json` が無い等で `HttpMessageNotReadableException`（400）。Boot3 なので DTO は `jakarta.validation` 注釈で固める。recordのフィールド名とJSONキーを一致させる（または `@JsonProperty`）。
- **Entityを直返しして事故る**：JPA Entity をそのまま返すと、**遅延ロードの巻き込み（N+1・LazyInitializationException）**、不要な内部項目の漏洩、双方向参照での無限ループ等が起きる。必ず DTO に詰め替える。→ [dto.md](./dto.md)
- **Controllerに業務ロジックを書きすぎる（Fat Controller）**：分岐・計算・複数リポジトリ操作が増えたら Service へ。Controller は「受けて・呼んで・返す」だけに保つ。
- **`@PathVariable` 名の不一致**：`@GetMapping("/{id}")` に対し引数名が異なる場合は `@PathVariable("id") Long userId` と明示する（コンパイル時に変数名が消えると解決できないため）。
- **例外を握りつぶして全部200で返す**：エラーは適切なステータス（404/400/409…）で返す。共通ハンドリングは `@ControllerAdvice` に集約。

## 関連
[service.md](./service.md) / [dto.md](./dto.md) / [validation.md](./validation.md)
