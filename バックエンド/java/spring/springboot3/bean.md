# Bean化（Spring Boot 3）

> ここは「Bean化」を**平易に1つずつ**。詳しい全体像（IoC/DI/スコープ/Qualifier等）は [ioc_di.md](./ioc_di.md)。

## ひとことで言うと
クラスを **「Spring（コンテナ）に管理してもらう状態にする」** こと。
Bean化すると、**自分で `new` しなくても、Springがそのインスタンスを作って、必要な場所に渡してくれる**。
コンテナに管理されている1個1個のオブジェクトを **Bean** と呼ぶ。

## ステップ1：Bean化しないと何が困るのか
自分で組み立てると、依存が連鎖して大変になる:
```java
// 全部 自分で new する世界
var repo    = new UserRepository(dataSource);
var gateway = new StripeGateway(httpClient, apiKey);
var service = new UserService(repo, gateway);   // ← これを使いたいだけなのに、下を全部組む
```
- 使いたい1個のために、その下の部品まで全部手で組む。
- しかも `StripeGateway` を別実装に変えたら、`new` してる所を全部直す。
- さらに（重要）**`new` で作ったものには Springの機能（後述）が効かない**。

## ステップ2：Bean化のやり方（2通り）
### ① 自分のクラス → アノテーションを付ける
クラスに目印（アノテーション）を付けるだけ。起動時にSpringが見つけて登録（component scan）。
```java
@Service                       // ← これが「Bean化」の目印
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) {   // 必要な相手を書くだけ
        this.repo = repo;                       // ← Springが勝手に渡してくれる
    }
}
```
目印の種類（役割で使い分けるが、**本質はどれも同じ＝Bean化**）:
```
@Component         … 汎用
@Service           … 業務ロジック
@Repository        … DBアクセス
@Controller / @RestController … Web入口
```

### ② 外部ライブラリのクラス → `@Bean` メソッドで登録
自分でアノテーションを付けられない（他人が作った）クラスは、`@Configuration` の中で `@Bean` を書いて登録:
```java
@Configuration
class AppConfig {
    @Bean
    RestClient restClient() {           // 外部ライブラリのクラスをBean化
        return RestClient.create();
    }
}
```
→ 詳しくは [beans_config.md](./beans_config.md)

## ステップ3：Bean化すると何が起きるか
1. **Springが起動時にインスタンスを1個作る**（既定でアプリ内1個＝シングルトン）。
2. それを**必要とする場所（コンストラクタ）に自動で渡す**（＝DI / 依存性注入）。
3. **Springの機能が効くようになる**：`@Transactional`（トランザクション）、`@Async`、`@Cacheable`、AOP など。
   → これらは「コンテナが管理しているBeanだから」かけられる仕掛け。

```
 Bean化したクラス  ──登録──►  Springコンテナ（管理リスト）
                                  │ 必要な所へ自動で配る（DI）
                                  ▼
                        UserService ← UserRepository が注入された状態で渡される
```

## ステップ4：Bean化しない（自分で new する）とどうなる
```java
var service = new UserService(repo);   // ただのオブジェクト。コンテナの管理外
```
- `@Transactional` が **効かない**（トランザクションが張られない）。
- `@Autowired` も効かず、依存が **null** になりがち。
- → **Beanは必ずコンテナ経由で受け取る**（自分でnewしない）のが鉄則。

## ハマりどころ
- **目印の付け忘れ** → Beanにならず `NoSuchBeanDefinitionException`。
- **component scanの範囲外**にクラスを置く（mainクラスのpackageより外）→ 見つからない。mainはルートpackageへ。→ [getting_started.md](./getting_started.md)
- **自分で new してしまう** → コンテナ管理外で `@Transactional` 等が無効。
- **同じ型のBeanが複数** → どれを注入するか決められず `NoUniqueBeanDefinitionException`。`@Primary`/`@Qualifier` で明示。→ [ioc_di.md](./ioc_di.md)

## 一言まとめ
> **Bean化＝「このクラス、Springに管理してもらう」と目印を付けること。**
> すると Springが作って・配って（DI）・機能（@Transactional等）を効かせてくれる。自分でnewするとそれが全部効かない。

## 関連
[ioc_di.md](./ioc_di.md)（DI全体・詳しく）/ [beans_config.md](./beans_config.md)（@Bean登録）/ [service.md](./service.md)
