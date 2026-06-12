# 実務でハマる罠まとめ（Pitfalls）（Spring Boot 3）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

> **この版 = Spring Boot 3 系**（Spring Framework 6 / Spring Security 6 / Java 17+、`jakarta.*`）。2系（`javax.*` / Security 5）とは挙動・APIが変わる点が多い。

## 役割・なぜ必要か
- Springは「アノテーション1個で動く」一方、**プロキシ・DIコンテナ・JPAの遅延読み込み**など見えない仕組みが事故の温床。症状から該当箇所へ素早く飛ぶための索引。

## Boot 3 移行で最初に踏む地雷
- **`javax.*` → `jakarta.*` 移行**：Boot3最大の破壊的変更。`javax.persistence`→`jakarta.persistence`、`javax.servlet`→`jakarta.servlet`、`javax.validation`→`jakarta.validation` 等。import が古いままだとコンパイル不能。ライブラリも Jakarta 対応版が必要。→ [getting_started.md](./getting_started.md) / [entity_jpa.md](./entity_jpa.md)
- **`WebSecurityConfigurerAdapter` 廃止**（Security 6）：継承する旧スタイルは削除済み。`SecurityFilterChain` を **Bean** として定義する関数型スタイルへ。`authorizeRequests`→`authorizeHttpRequests` 等もリネーム。→ [security.md](./security.md)
- **Spring Framework 6 / Java 17 ベースライン**：Java 8/11 ではそもそも起動しない。古いライブラリ・古いアノテーション前提のコードは要更新。→ [getting_started.md](./getting_started.md)

## DI / Bean
- **フィールドインジェクション（`@Autowired` をフィールドに）非推奨**：テスト困難・不変にできない・循環依存が隠れる。**コンストラクタ注入**（`final` フィールド＋Lombok `@RequiredArgsConstructor`）が標準。→ [ioc_di.md](./ioc_di.md)
- **`new` で自分でインスタンス化**：DIコンテナ管理外になり `@Transactional`・`@Async`・AOPが一切効かない。Beanとして注入して使う。→ [ioc_di.md](./ioc_di.md) / [beans_config.md](./beans_config.md)
- **同じ型のBeanが複数で起動失敗**：`NoUniqueBeanDefinitionException`。`@Primary` か `@Qualifier` で解決。→ [beans_config.md](./beans_config.md)
- **循環依存**：A→B→A の相互注入。設計を見直す（中間Beanに切り出す）。コンストラクタ注入なら起動時に検出できる。→ [ioc_di.md](./ioc_di.md)

## JPA / Entity / DB
- **N+1問題**：`LAZY` 関連を一覧ループで触ると関連ごとにSQLが `1+N` 回飛ぶ。**fetch join**（JPQL `join fetch`）か `@EntityGraph` でまとめ読み。Hibernateのログでクエリ数を確認。→ [entity_jpa.md](./entity_jpa.md)
- **EntityをそのままJSONで返す**：(1) Controller応答時にトランザクションが切れていると `LazyInitializationException`、(2) パスワード等の内部項目がそのまま漏れる、(3) 双方向関連で無限ループ。**DTOへ詰め替えて返す**のが鉄則。→ [dto.md](./dto.md) / [entity_jpa.md](./entity_jpa.md)
- **`open-in-view` に依存**：デフォルト有効でView層まで遅延読込が効くため上記N+1/Lazyを覆い隠す。本番は無効化し、Service内で必要分をfetchしておく。→ [entity_jpa.md](./entity_jpa.md)
- **双方向関連のJSON無限ループ**：`@OneToMany`↔`@ManyToOne` を直接シリアライズすると循環。DTO化 or `@JsonIgnore` / `@JsonManagedReference`。→ [dto.md](./dto.md) / [entity_jpa.md](./entity_jpa.md)
- **`save()` で全カラムUPDATE / 不要なSELECT**：JPAの挙動を理解せず使うと無駄なクエリ。更新は必要なフィールドだけ、バッチは `saveAll` やJPQLで。→ [repository.md](./repository.md)
- **`ddl-auto: update` を本番で使う**：スキーマが暗黙変更され事故。本番は `validate` か `none`、変更はマイグレーション（Flyway/Liquibase）で。→ [config_properties.md](./config_properties.md)

## トランザクション（@Transactional）
- **自己呼び出しが効かない**：同一クラス内で `this.methodA()` から `@Transactional` な `methodB()` を呼ぶと**プロキシを経由せず**注釈が無視される。別Beanに分ける／自己プロキシ注入で回避。→ [transactions.md](./transactions.md) / [aop.md](./aop.md)
- **checked例外でロールバックされない**：デフォルトは **`RuntimeException` と `Error` のみロールバック**。検査例外を投げてもコミットされる。`@Transactional(rollbackFor = Exception.class)` を明示。→ [transactions.md](./transactions.md)
- **例外をcatchして握り潰す**：トランザクション内で例外を飲み込むとロールバック契機を失う。あるいは `rollback-only` マークだけ残り `UnexpectedRollbackException`。→ [transactions.md](./transactions.md)
- **`private` / `final` メソッドに付与**：プロキシが被せられず無効。`public` な非finalメソッドに付ける。→ [transactions.md](./transactions.md) / [aop.md](./aop.md)
- **トランザクションを跨ぐ重い処理**：外部API呼び出し等を `@Transactional` 内に入れるとDBコネクションを長時間占有。トランザクション境界は最小限に。→ [transactions.md](./transactions.md)

## AOP / 非同期
- **`@Async` の自己呼び出しが効かない**：`@Transactional` と同じくプロキシ経由でないと無視され、**同期実行**される。別Beanに切り出す。→ [async_scheduling.md](./async_scheduling.md) / [aop.md](./aop.md)
- **`@EnableAsync` / `@EnableScheduling` の付け忘れ**：注釈だけ書いても有効化されない。→ [async_scheduling.md](./async_scheduling.md)
- **`@Async` の戻り値を `void` で例外握り潰し**：別スレッドの例外が表に出ない。`CompletableFuture` を返すか `AsyncUncaughtExceptionHandler` を設定。→ [async_scheduling.md](./async_scheduling.md)
- **デフォルトスレッドプールのまま本番投入**：`@Async` のExecutor未設定だと毎回スレッド生成で枯渇しうる。`ThreadPoolTaskExecutor` を明示。→ [async_scheduling.md](./async_scheduling.md)

## セキュリティ / Actuator
- **`SecurityFilterChain` のマッチ順ミス**：`permitAll` と `authenticated` の記述順・パスパターンを誤ると全公開／全遮断。→ [security.md](./security.md)
- **CSRF設定の取り違え**：REST APIで無効化すべき場面・有効にすべき場面を混同。トークン認証APIとフォーム認証で扱いが違う。→ [security.md](./security.md)
- **Actuatorエンドポイントの公開しすぎ**：`management.endpoints.web.exposure.include=*` で `/actuator/env` `/heapdump` 等が外部公開され情報漏洩。必要なものだけ公開し、認証で保護。→ [actuator.md](./actuator.md) / [security.md](./security.md)
- **パスワードを平文 / 弱いエンコーダで保存**：`PasswordEncoder`（BCrypt等）を必ず通す。→ [security.md](./security.md)

## 設定 / プロファイル
- **プロファイル有効化漏れ**：`application-prod.yml` を作っても `spring.profiles.active=prod` を渡し忘れて**デフォルト設定で本番起動**。環境変数 `SPRING_PROFILES_ACTIVE` 等で明示。→ [config_properties.md](./config_properties.md)
- **DBパスワード等のハードコード**：`application.yml` に直書きせず環境変数／Secret管理で外出し。→ [config_properties.md](./config_properties.md)
- **`@Value` のキー名タイポ**：解決できず空文字やnull注入。型安全な `@ConfigurationProperties` を優先。→ [config_properties.md](./config_properties.md)

## 設計（層分け）
- **Fat Controller / Fat Service**：Controllerに業務ロジック、あるいはServiceが肥大化。Controllerは入出力変換に徹し、業務はService、複雑なら別クラスへ分割。→ [service.md](./service.md) / [controller.md](./controller.md)
- **Controllerで例外を都度try-catch**：分散すると重複。`@RestControllerAdvice`（`@ControllerAdvice`）で集約ハンドリング。→ [exception_handling.md](./exception_handling.md)
- **バリデーションをService内に手書き**：入力検証は `@Valid` ＋ Bean Validation（`jakarta.validation`）で境界に寄せる。→ [validation.md](./validation.md)

## テスト
- **`@SpringBootTest` 乱用で遅い**：毎テストでコンテキスト全起動。用途に応じ **スライステスト**（`@WebMvcTest`＝Controller層、`@DataJpaTest`＝JPA層）に分け、純粋ロジックはMockito単体テストへ。→ [testing.md](./testing.md)
- **コンテキストキャッシュを壊す設定**：`@MockBean` や `@TestPropertySource` を多用するとキャッシュが効かず毎回再起動して激遅。設定を揃える。→ [testing.md](./testing.md)
- **本物のDBに依存したテスト**：環境差で不安定。**Testcontainers**（Boot 3.1+連携）で本番同等DBを使い捨て起動。→ [testing.md](./testing.md)

## 関連
[ioc_di.md](./ioc_di.md) / [entity_jpa.md](./entity_jpa.md) / [transactions.md](./transactions.md) / [async_scheduling.md](./async_scheduling.md) / [security.md](./security.md) / [actuator.md](./actuator.md) / [dto.md](./dto.md) / [testing.md](./testing.md)
