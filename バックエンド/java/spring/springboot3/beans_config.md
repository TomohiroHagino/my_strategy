# Bean定義 / @Configuration / 自動設定（Spring Boot 3）

## ひとことで言うと
**DIコンテナに「どんな部品（Bean）を、どう作るか」を教える仕組み**。`@Configuration`＋`@Bean` で手動定義でき、Spring Boot では **auto-configuration（自動設定）** がスターター（依存）に応じて大量のBeanを勝手に組み立ててくれる。

## 役割・なぜ必要か
- アプリは多数のオブジェクト（DataSource、ObjectMapper、各種クライアント…）の集合体。これらを **「いつ・どう生成し・誰に渡すか」** を一元管理するのがBean定義。
- `@Component` / `@Service` 等の **コンポーネントスキャン（自分のクラス）** に対し、`@Bean` は **自分で new を制御したい / ライブラリ提供クラスをBean化したい** ときに使う（ライブラリ側のクラスにはアノテーションを足せないため）。
- Spring Boot の肝が **auto-configuration**。`spring-boot-starter-web` を入れるだけで Tomcat・DispatcherServlet・Jackson などが自動構成される。「設定ゼロで動く」体験の正体がこれ。

## 基本の書き方（コード）
```java
// 1) 手動Bean定義：ライブラリのクラスをBean化する典型例
@Configuration
public class AppConfig {

    // メソッド名がBean名（既定）。戻り値の型でDIされる
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
                .baseUrl("https://api.example.com")
                .build();
    }

    // 他Beanへの依存は「引数」で受け取る（DIコンテナが解決して渡す）
    @Bean
    public OrderClient orderClient(RestClient restClient) {
        return new OrderClient(restClient);
    }
}
```

```java
// 2) @Value で application.yml / 環境変数の値を注入
@Component
public class MailSender {
    private final String from;
    private final int timeoutMs;

    // ${...} はプロパティ参照、: の後ろは既定値
    public MailSender(
            @Value("${app.mail.from}") String from,
            @Value("${app.mail.timeout-ms:3000}") int timeoutMs) {
        this.from = from;
        this.timeoutMs = timeoutMs;
    }
}
```

```java
// 3) Beanスコープ：既定は singleton（アプリ全体で1個を共有）
@Bean
@Scope("prototype")   // 取得のたびに新インスタンス（状態を持つ部品向け）
public ReportBuilder reportBuilder() {
    return new ReportBuilder();   // singletonなら全リクエストで共有される点に注意
}
```

## 実務での使い方・定番パターン
- **自前クラスは `@Service` / `@Repository` / `@Component`**、**ライブラリ製クラスや new に分岐が要る物は `@Bean`** と使い分ける。
- **条件付きBean**：`@ConditionalOnXxx` で「ある条件のときだけ」Beanを作る。auto-configuration の中核技術。
```java
@Configuration
public class CacheConfig {

    // ユーザがCacheManager Beanを定義していない時だけ既定を提供
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users");
    }

    // application.yml に app.feature.cache=true がある時だけ有効
    @Bean
    @ConditionalOnProperty(name = "app.feature.cache", havingValue = "true")
    public CacheWarmer cacheWarmer() {
        return new CacheWarmer();
    }
}
```
- **自動設定の「上書き」**：自分で同じ型のBeanを定義すると、auto-configuration 側は `@ConditionalOnMissingBean` のため作らない＝あなたの定義が優先される。これが Spring Boot の「規約だが必要なら差し替え可」の正体。
- **自動設定の「無効化」**：効かせたくない自動構成は除外できる。
```java
// 例：DataSource の自動設定を切る（接続情報を後から動的に組む等）
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class App { }
```
- **何が自動構成されたか調べる**：`--debug` 起動、または Actuator の `/actuator/conditions` で「適用された / されなかった」条件レポートが見られる。詰まったらまずここ。
- 設定値はバラ撒かず **`@ConfigurationProperties`** で型安全にまとめるのが定石。→ [config_properties.md](./config_properties.md)

## ハマりどころ / アンチパターン
- **singleton に可変状態を持たせる（最頻出の事故）**：`@Service` や `@Bean`（既定 singleton）は全リクエストで**1インスタンス共有**。インスタンスフィールドに「処理中のユーザ」等を持つと**他リクエストと混線**する。状態はメソッド引数・ローカル変数に閉じ、フィールドは不変（final 依存のみ）に保つ。
- **自動設定の理解不足で「上書きしたつもりが効かない / 二重定義」**：自動構成は多くが `@ConditionalOnMissingBean`。同型Beanを1個だけ定義すれば差し替わるが、**条件や型がズレる**と自動Beanが残り想定外の挙動に。`/actuator/conditions` で実際の適用を確認する。
- **`@Bean` メソッドを普通の Java 呼び出しのように直接呼ぶ**：`@Configuration` クラス内なら Spring が割り込んで singleton を返すが、**仕組みを知らずに別クラスから new** すると DI も singleton 保証も外れる。Beanは「注入で受け取る」が原則。
- **`@Value` のキー打ち間違い / 既定値なし**：該当キーが無いと起動時に解決失敗で落ちる。任意項目は `:既定値` を付ける。型不一致（数値に文字列）も起動時エラー。
- **prototype を singleton に注入して「1個しか作られない」**：singleton Bean のフィールドに prototype を注入すると、注入は**生成時の1回だけ**。毎回新インスタンスが欲しいなら `ObjectProvider<T>` や Lookup で都度取得する。

## 関連
[ioc_di.md](./ioc_di.md) / [config_properties.md](./config_properties.md)
