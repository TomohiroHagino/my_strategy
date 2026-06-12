# IoC / DI / Bean（Spring Boot 3）

> 「Bean化って何？」をまず平易に知りたいなら → [bean.md](./bean.md)（入口）。ここはその詳しい全体版。

## ひとことで言うと
**IoC（制御の反転）** ＝「オブジェクトの生成と結合を、自分（new）ではなく **DIコンテナ（ApplicationContext）** に任せる」こと。コンテナが管理する1つ1つのオブジェクトが **Bean**。必要な相手をコンテナが差し込むのが **DI（依存性注入）**。Springの全機能はこの上に乗る、最大の肝。

## 役割・なぜ必要か
- 自分で `new UserService(new UserRepository(...))` と書くと、依存が**ハードコード**され、差し替え・テストが困難になる。
- DIなら **「何が必要か」だけ宣言**し、**「どれを使うか」はコンテナが決める**。これで以下が手に入る：
  - **疎結合**：ServiceはRepositoryの「型（interface）」だけに依存。実装が変わっても無傷。
  - **差し替え**：本番はJPA実装、テストはモック、と注入物を入れ替えるだけ。
  - **テスト容易**：依存をコンストラクタから渡せるので、コンテナ無しでも単体テストできる。
- 起動時、コンテナは **component scan** でBeanを集め、依存グラフを解いて結線し、シングルトンとして保持する。

## 基本の書き方（コード）
**ステレオタイプアノテーション**でクラスをBean登録する（役割で使い分けるが、本質はどれも `@Component`）。
```java
@Component   // 汎用のBean
@Service     // 業務ロジック層（意味づけ）
@Repository   // DBアクセス層（＋DB例外を共通例外に変換）
@Controller / @RestController  // Web層
```
**コンストラクタ注入（推奨）**。`final` で不変、必須依存が型で保証される。
```java
// service/OrderService.java
import org.springframework.stereotype.Service;

@Service
public class OrderService {
    private final PaymentGateway gateway;     // 依存（interface）
    private final OrderRepository repo;

    // コンストラクタが1つなら @Autowired は省略可（Springが自動で注入）
    public OrderService(PaymentGateway gateway, OrderRepository repo) {
        this.gateway = gateway;
        this.repo = repo;
    }
}
```
注入される側もBeanにしておく（component scanで拾われる）。
```java
public interface PaymentGateway { void charge(long amount); }

@Component
class StripeGateway implements PaymentGateway {
    public void charge(long amount) { /* ... */ }
}
```
**複数候補があるとき**は `@Qualifier`（名前指定）か `@Primary`（既定指定）で曖昧さを解消。
```java
@Component @Primary                 // 既定で選ばれる
class StripeGateway implements PaymentGateway { /* ... */ }

@Component("paypal")
class PaypalGateway implements PaymentGateway { /* ... */ }

@Service
class CheckoutService {
    private final PaymentGateway gateway;
    // 名前で明示指定 → PaypalGatewayが入る
    public CheckoutService(@Qualifier("paypal") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```
**スコープ**（既定はsingleton：アプリで1個）。
```java
@Component
@Scope("prototype")  // 取得のたびに新インスタンス（必要なときだけ）
class RequestId { /* ... */ }
```

## 実務での使い方・定番パターン
- **コンストラクタ注入で統一**する。テストで `new OrderService(mockGateway, mockRepo)` がそのまま書け、依存の必須性が型で明確。`final` も付けられる。
- **依存の数が増えたら設計を疑う**：コンストラクタ引数が5個6個になったら、そのクラスが多くを抱えすぎ（責務過多）のサイン。分割する。
- **interfaceに依存させる**と差し替えが効く（本番実装 / テスト用スタブ / 別ベンダ実装）。ただし実装が1つしかない段階で機械的にinterface化するのはYAGNI。
- **`@Bean`メソッドでの登録**は、外部ライブラリのクラス（自分でアノテーションを付けられない）をBean化するときに使う。→ [beans_config.md](./beans_config.md)
- **テスト**では `@MockBean`/`@MockitoBean` でコンテナ内のBeanをモックに差し替えられる。→ [testing.md](./testing.md)
- **設定値の注入**は `@Value("${...}")` や `@ConfigurationProperties` で。直書きせずDIに乗せる。→ [config_properties.md](./config_properties.md)

## ハマりどころ / アンチパターン
- **field injection（`@Autowired private Foo foo;`）は非推奨**：
  - `final` にできず不変性が崩れる。
  - コンストラクタを通らないので **テストでモックを差し込めない**（リフレクションが要る）。
  - 依存がコンストラクタに現れず、隠れた依存が増えても気づけない。
  → コンストラクタ注入に統一する。
- **循環依存**：A→B かつ B→A をコンストラクタ注入すると起動時に `BeanCurrentlyInCreationException`。設計の歪みのサインなので、共通処理を第3のBeanへ抽出して一方向にする（`@Lazy` での回避は応急処置）。
- **Bean候補が複数で `NoUniqueBeanDefinitionException`**：同じ型のBeanが2つ以上あると注入先を決められない。`@Primary` か `@Qualifier` で明示する。
- **Beanが見つからず `NoSuchBeanDefinitionException`**：ステレオタイプの付け忘れ、または **component scan範囲外**（mainクラスのpackageより外）にクラスを置いた。mainはルートpackageへ。→ [getting_started.md](./getting_started.md)
- **自分でnewしてしまう**：`new UserService()` で作ったインスタンスはコンテナ管理外。`@Transactional` や `@Autowired` が一切効かず、依存もnullになる。Beanは必ずコンテナ経由で取得する。
- **prototypeをsingletonに注入**：singleton側は生成時に1度だけ受け取るので、毎回新しくはならない。スコープ混在の挙動を理解して使う。

## 関連
[beans_config.md](./beans_config.md) / [service.md](./service.md) / [project_structure.md](./project_structure.md)
