# プロジェクトの始め方（Spring Boot 3）

## ひとことで言うと
Spring Boot は **「設定より規約」でSpringアプリを即起動できる土台**。雛形は **Spring Initializr**（[start.spring.io](https://start.spring.io)）で生成し、**組込Tomcat入りの実行可能jar**として `main` 一発で立ち上がる。Boot 3 は **Java 17 が必須**（21対応）。

## 役割・なぜ必要か
- 素のSpring Frameworkは設定（XML/Java Config）が膨大。Bootは **starter依存（部品の詰め合わせ）＋自動設定（auto-configuration）** で、定型のボイラープレートを消す。
- **組込サーバ**（Tomcat/Jetty/Undertow）を内包するので、外部のアプリサーバへwarをデプロイせず、`java -jar app.jar` で動く（コンテナ化と相性◎）。
- **Boot 3 = Jakarta EE 移行**が最重要。インポートは `javax.*` ではなく **`jakarta.*`**（例: `jakarta.persistence.Entity`）。2系の資産を持ち込むと一番ここで詰まる。

## 基本の書き方（コード）
Initializrで生成（CLIでも可）。Spring Web / Spring Data JPA / H2 などのstarterを選ぶ。
```bash
# curlでInitializrから雛形zipを取得（Maven版）
curl https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java \
  -d bootVersion=3.3.0 -d javaVersion=17 \
  -d dependencies=web,data-jpa,h2 \
  -d groupId=com.example -d artifactId=demo \
  -o demo.zip && unzip demo.zip -d demo
```
依存（Maven `pom.xml` の抜粋）。starterを足すだけでWeb/JPAが揃う。
```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.3.0</version>
</parent>
<properties><java.version>17</java.version></properties>
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>   <!-- 組込Tomcat込み -->
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
</dependencies>
```
mainクラス。`@SpringBootApplication` 1枚で「設定クラス＋自動設定＋component scan」を兼ねる。
```java
// src/main/java/com/example/demo/DemoApplication.java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args); // ここでDIコンテナ起動＆Tomcat起動
    }
}
```
動作確認用の最小コントローラ。
```java
// src/main/java/com/example/demo/HelloController.java
package com.example.demo;

import org.springframework.web.bind.annotation.*;

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() { return "Hello Spring Boot 3"; }
}
```
設定ファイル。`application.properties` でも良いが **`application.yml`** が実務の主流。
```yaml
# src/main/resources/application.yml
server:
  port: 8080            # 組込Tomcatの待受ポート
spring:
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    hibernate:
      ddl-auto: update   # 開発用。本番はFlyway/Liquibaseでマイグレーション
```

起動とビルド（`mvnw`/`gradlew` はリポジトリ同梱のwrapper。各自のローカルにMaven/Gradle未導入でも動く）。
```bash
./mvnw spring-boot:run        # 開発起動（Maven）
./gradlew bootRun             # 開発起動（Gradle）
./mvnw clean package          # 実行可能jarを生成 → target/demo-0.0.1-SNAPSHOT.jar
java -jar target/demo-0.0.1-SNAPSHOT.jar   # 本番同等の起動
```

## 実務での使い方・定番パターン
- **starterは足し算**で考える。Web→`web`、DB→`data-jpa`＋ドライバ、バリデーション→`validation`、運用監視→`actuator`、認証→`security`。必要になってから足す。
- **プロジェクト構成**は package by layer（`controller` / `service` / `repository` / `entity` / `dto`）が基本形。→ [project_structure.md](./project_structure.md)
- **DevTools**（`spring-boot-devtools`）で保存時の自動再起動。開発体験が大きく変わる。
- **プロファイル**で環境を切り替え（`application-dev.yml` / `application-prod.yml`、`--spring.profiles.active=prod`）。→ [config_properties.md](./config_properties.md)
- **wrapper（`mvnw`/`gradlew`）をコミット**して、CIや他メンバーとビルドツールのバージョンを固定する。
- 起動時に出る **ASCIIバナー**と「Started DemoApplication in X seconds」で、コンテナ初期化が成功したか即わかる。

## ハマりどころ / アンチパターン
- **Javaバージョン不一致**：Boot 3 は **Java 17未満で起動不可**。`java -version` とIDEのProject SDK、`pom.xml` の `java.version` を全部17（以上）に揃える。古いJDKだと意味不明なクラスロードエラーになる。
- **`port 8080` 衝突**：`Web server failed to start. Port 8080 was already in use.` → 既存プロセスを落とすか `server.port` を変更（`./mvnw spring-boot:run -Dspring-boot.run.arguments=--server.port=8081`）。
- **starter選択ミス**：DBドライバ（H2/PostgreSQL等）を入れ忘れて `data-jpa` だけ入れると、DataSource自動設定で起動失敗。逆にWeb系が不要なバッチで `web` を入れると無駄にTomcatが立つ。
- **`javax.*` を書いてしまう**：Boot 3 では `jakarta.*` が正。IDEの自動importが古い候補を出すことがある。→ [pitfalls.md](./pitfalls.md)
- **mainクラスをpackageの末端に置く**：component scanは `@SpringBootApplication` のあるpackage配下が対象。これより外側にBeanを置くと「Beanが見つからない」になる。ルートpackageに置くのが鉄則。
- **`ddl-auto: update` を本番運用**：開発の便利機能。本番はスキーマを破壊しうるのでマイグレーションツールに任せる。

## フォルダ構成（始動直後）

> Spring Initializr が生成するのは「起動クラス・テスト雛形・resources・ビルド系」だけ。
> **層（controller/service/repository/…）と中のクラスは自分で作る**。下は典型的な構築後の姿。

```
demo/
├── src/
│   ├── main/
│   │   ├── java/com/example/demo/
│   │   │   ├── DemoApplication.java             # @SpringBootApplication・main()【Initializr生成】
│   │   │   │
│   │   │   ├── controller/                      # ── ここから下は自分で作る ──
│   │   │   │   └── UserController.java           # @RestController（受付・HTTP）
│   │   │   ├── service/
│   │   │   │   ├── UserService.java              # @Service（業務ロジックのIF）
│   │   │   │   └── impl/
│   │   │   │       └── UserServiceImpl.java      # その実装
│   │   │   ├── repository/
│   │   │   │   └── UserRepository.java           # JpaRepository（DBアクセス）
│   │   │   ├── entity/                           # ＝ domain/ と呼ぶことも
│   │   │   │   └── User.java                     # @Entity（テーブル対応・jakarta.persistence）
│   │   │   ├── dto/
│   │   │   │   ├── UserRequest.java              # 入力（@Valid でバリデーション）
│   │   │   │   └── UserResponse.java             # 出力（Entityを直接返さない）
│   │   │   ├── mapper/                           # Entity↔DTO 変換（MapStruct等）
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java           # @Configuration（Spring Security）
│   │   │   │   └── WebConfig.java                # CORS / Interceptor
│   │   │   └── exception/
│   │   │       ├── GlobalExceptionHandler.java   # @RestControllerAdvice（例外を集約）
│   │   │       └── NotFoundException.java
│   │   └── resources/
│   │       ├── application.yml                   # 共通設定（or application.properties）【Initializr生成】
│   │       ├── application-dev.yml               # 開発プロファイル（自分で作る）
│   │       ├── application-prod.yml              # 本番プロファイル（自分で作る）
│   │       ├── static/                           # 静的ファイル（CSS/JS/画像）【生成】
│   │       ├── templates/                        # Thymeleaf 等のテンプレート【生成】
│   │       ├── messages.properties               # i18n メッセージ（自分で作る）
│   │       └── db/migration/                     # Flyway（自分で作る）
│   │           └── V1__init.sql                  # 初期スキーマ
│   └── test/
│       └── java/com/example/demo/
│           ├── DemoApplicationTests.java         # @SpringBootTest（起動テスト）【Initializr生成】
│           ├── controller/
│           │   └── UserControllerTest.java       # @WebMvcTest（自分で作る）
│           └── service/
│               └── UserServiceTest.java          # Mockito 単体テスト（自分で作る）
│
├── build.gradle                                 # 依存・プラグイン定義（Mavenなら pom.xml）【生成】
├── settings.gradle                               # プロジェクト名【生成】
├── gradlew / gradlew.bat                          # Gradle wrapper（実行スクリプト）【生成】
├── gradle/wrapper/
│   ├── gradle-wrapper.jar
│   └── gradle-wrapper.properties                 # Gradleのバージョン固定【生成】
├── mvnw / mvnw.cmd + .mvn/wrapper/                # Maven選択時の wrapper【生成】
├── compose.yaml                                  # 任意：DB等を Docker Compose で（Initializrで選択可）
├── Dockerfile                                    # 任意：jar 1個をコンテナ化
├── .gitignore                                    # 【生成】
├── HELP.md                                       # Initializr のヘルプ【生成】
└── README.md
```
- **【生成】= Spring Initializr が作る**：起動クラス・テスト雛形・`resources`（application.properties・static・templates）・ビルド系（build/wrapper・.gitignore・HELP.md）。
- **層（controller/service/repository/entity/dto/config/exception）と中のクラスは自分で作る**（package by layer）。`mapper/` は MapStruct 等を使う場合。
- 設定は `application.yml` を共通とし、`application-{dev,prod}.yml` でプロファイル分け（`--spring.profiles.active=prod` で切替）。
- 本番のスキーマ変更は `ddl-auto` ではなく `db/migration/`（Flyway）で管理するのが定番。

## 関連
[project_structure.md](./project_structure.md) / [config_properties.md](./config_properties.md) / [ioc_di.md](./ioc_di.md)
