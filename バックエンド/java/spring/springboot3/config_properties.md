# 設定 / プロファイル（Spring Boot 3）

## ひとことで言うと
アプリの**設定値**（DB接続先・ポート・機能フラグ等）を `application.yml` / `application.properties` に外出しし、`@Value` や `@ConfigurationProperties` で読み込む仕組み。**プロファイル**で環境（dev / prod 等）ごとに切り替える。

## 役割・なぜ必要か
- 接続先やログレベルをコードに直書きすると、環境を変えるたびにビルドし直しになる。**設定を外部化**することで、同じ成果物を環境変数や設定ファイルだけで切り替えられる。
- 「開発はローカルDB・詳細ログ、本番は本番DB・最小ログ」のような切り替えを、**プロファイル**で宣言的に分けられる（Rails の `config/environments/*.rb` に相当）。
- 機密（パスワード・APIキー）をソースに残さず、**環境変数や外部の秘密管理**から注入できる。
- 設定値は**型安全に束ねて扱う**ほうが安全。バラバラの `@Value` より `@ConfigurationProperties` クラスにまとめるのが Boot3 の定番。

## 基本の書き方（コード）
```yaml
# application.yml（共通設定。${ENV:デフォルト} で環境変数を参照できる）
server:
  port: 8080
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/app}
    username: ${DB_USER:app}
    password: ${DB_PASSWORD}        # 機密はデフォルトを置かず、環境変数必須にする
app:
  feature:
    new-checkout: false
  api:
    base-url: https://api.example.com
    timeout-ms: 3000
```

```java
// @Value … 単一の値を直接注入（手軽だが散らばりやすい）
import org.springframework.beans.factory.annotation.Value;

@Service
public class CheckoutService {
    @Value("${app.feature.new-checkout}")
    private boolean newCheckoutEnabled;
}
```

```java
// @ConfigurationProperties … 関連する設定を型安全に束ねる（推奨）
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app.api")
public record ApiProperties(
    String baseUrl,        // app.api.base-url（kebab → camel に自動マッピング）
    int timeoutMs
) {}

// 有効化：起動クラスに @ConfigurationPropertiesScan を付けるか
//        @EnableConfigurationProperties(ApiProperties.class) を指定
@SpringBootApplication
@ConfigurationPropertiesScan
public class App { public static void main(String[] a){ SpringApplication.run(App.class,a);} }

// 使う側は普通に DI
@Service
public class ApiClient {
    private final ApiProperties props;
    public ApiClient(ApiProperties props) { this.props = props; }
}
```

## 実務での使い方・定番パターン
- **プロファイル別ファイル**：`application-dev.yml` / `application-prod.yml` を置き、`spring.profiles.active` で選ぶ。共通は `application.yml`、差分だけ各プロファイルに書く。
  ```yaml
  # application.yml に既定プロファイルを書ける
  spring:
    profiles:
      active: ${SPRING_PROFILES_ACTIVE:dev}
  ```
  ```yaml
  # application-prod.yml（本番だけの上書き）
  logging:
    level:
      root: WARN
  ```
- **プロファイルの有効化方法**（どれか）：
  ```bash
  # 環境変数（本番デプロイで定番）
  export SPRING_PROFILES_ACTIVE=prod
  # 起動引数
  java -jar app.jar --spring.profiles.active=prod
  # JVM プロパティ
  java -Dspring.profiles.active=prod -jar app.jar
  ```
- **1ファイルでプロファイルを分ける**：`---` で区切り `spring.config.activate.on-profile` を使う書き方もある（Boot2 の `spring.profiles` は Boot3 で非推奨）。
  ```yaml
  spring:
    config:
      activate:
        on-profile: dev
  logging:
    level:
      root: DEBUG
  ```
- **`@Profile` で Bean を切り替え**：`@Profile("prod")` を付けた Bean はそのプロファイル時だけ有効。モック実装と本番実装の切替に使う。→ [beans_config.md](./beans_config.md)
- **機密の外部化**：パスワード等は yaml に書かず、**環境変数**（インフラ/k8s Secret）や Vault などから注入する。Rails の credentials に相当する仕組みは外部に寄せるのが Boot の流儀。→ [security.md](./security.md)

## プロパティの優先順位（高い方が勝つ・抜粋）
```
1. コマンドライン引数（--server.port=9000）
2. OS 環境変数（SERVER_PORT=9000 … ドット/ハイフンは _ や大文字に）
3. application-{profile}.yml（プロファイル別）
4. application.yml（共通）
5. @ConfigurationProperties / コード上のデフォルト
```
> 「外側（起動時に渡すもの）ほど強い」と覚える。本番値は引数や環境変数で上書きする。

## ハマりどころ / アンチパターン
- **プロファイル有効化漏れ**：`SPRING_PROFILES_ACTIVE` を設定し忘れ、本番で dev 設定が動く。起動ログの `The following profiles are active:` を必ず確認。
- **機密を yaml にコミット**：パスワードや APIキーを `application.yml` に直書きして push する事故。**環境変数参照（`${...}`）**にして値はリポジトリ外へ。誤コミットしたら即ローテーション。→ [security.md](./security.md)
- **上書き順序の誤解**：「yaml に書いたのに反映されない」は、より優先度の高い環境変数や起動引数で上書きされているケースが多い。優先順位表を確認する。
- **`@Value` の乱用**：設定が増えると散らばって追えなくなる。関連値は `@ConfigurationProperties` に束ねる。
- **kebab/camel の取り違え**：`app.api.base-url` は `baseUrl` にマッピングされる（リラックスドバインディング）。プロパティ名のタイプミスは無言で無視されがちなので注意。
- **`@ConfigurationProperties` の有効化忘れ**：`@ConfigurationPropertiesScan` か `@EnableConfigurationProperties` が無いと Bean 登録されず注入できない。

## 関連: [beans_config.md](./beans_config.md) / [security.md](./security.md)
