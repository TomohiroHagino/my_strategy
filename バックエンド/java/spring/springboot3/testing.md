# テスト（JUnit5 / MockMvc / Testcontainers）（Spring Boot 3）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。Spring Boot 3 の標準は **JUnit 5 ＋ Mockito ＋ AssertJ**。「アプリの一部だけ起動する**スライステスト**」と「全部起動する `@SpringBootTest`」を使い分けるのが肝。

## 役割・なぜ必要か
- 変更のたびに全画面・全APIを手で確認するのは非現実的。**回帰（デグレ）を自動検出**するために要る。
- レイヤード構成（Controller/Service/Repository）なので、**層ごとに軽いテスト**を書けば速くて落ちにくいテスト群になる。
- `@SpringBootTest` で全Beanを立ち上げると遅い。**スライス**（Controllerだけ・Repositoryだけ）で必要な部分だけ起動するのが定番。

## 基本の書き方（コード）
```java
// 純粋な単体テスト：Springを起動せず Mockito だけ（最速・最多にすべき層）
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;          // 依存をモック化
    @InjectMocks OrderService service;   // モックを注入した実体

    @Test
    void 在庫不足なら例外を投げる() {
        // Arrange
        when(repo.stockOf(1L)).thenReturn(0);
        // Act / Assert（AssertJ）
        assertThatThrownBy(() -> service.place(1L))
            .isInstanceOf(OutOfStockException.class);
    }
}
```
```java
// @WebMvcTest：Controller層スライス。Service は @MockBean で差し替え、MockMvc でHTTPを検証
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;      // Controllerが依存するBeanはモック必須

    @Test
    void 一覧は200とJSONを返す() throws Exception {
        when(service.findAll()).thenReturn(List.of(new OrderDto(1L, "PAID")));
        mvc.perform(get("/orders"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$[0].status").value("PAID"));
    }
}
```
```java
// @DataJpaTest：Repository層スライス。既定は組込みH2＋テスト後ロールバック
@DataJpaTest
class OrderRepositoryTest {
    @Autowired OrderRepository repo;

    @Test
    void ステータスで検索できる() {
        repo.save(new Order("PAID"));
        assertThat(repo.findByStatus("PAID")).hasSize(1);
    }
}
```
```java
// @SpringBootTest + Testcontainers：実DB(PostgreSQL)で結合テスト（重いが本物に近い）
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderApiIT {
    @Container @ServiceConnection   // Boot 3.1+: 接続情報を自動注入（url/user/pass不要）
    static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");

    @Autowired TestRestTemplate rest;

    @Test
    void 注文を作成して取得できる() {
        var res = rest.postForEntity("/orders", new OrderDto(null, "NEW"), OrderDto.class);
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```
```yaml
# src/test/resources/application.yml — テスト専用プロファイル
spring:
  jpa:
    hibernate:
      ddl-auto: create-drop    # テスト用スキーマを毎回作り直す
```

## 実務での使い方・定番パターン
- **テストピラミッド**：土台に大量の高速な単体テスト（Mockito）、中間にスライス（`@WebMvcTest`/`@DataJpaTest`）、頂点に少数の `@SpringBootTest` 結合テスト。
- **スライスを使い分ける**：Controllerの入出力・バリデーション・ステータスは `@WebMvcTest`、JPAクエリ・マッピングは `@DataJpaTest`。それぞれ**最小Beanしか立ち上げず速い**。
- **`@MockBean` で境界を差し替え**：`@WebMvcTest` ではService、結合テストでは外部APIクライアントをモック化。DBは Testcontainers で本物を使う。
- **Testcontainers で本番同等DB**：H2 と PostgreSQL は方言差（JSONB・配列型・upsert）で挙動が違う。重要なクエリは**実DBで結合**して方言事故を防ぐ。`@ServiceConnection`（3.1+）で接続設定が自動化され定型コードが消える。
- **AssertJ で読みやすく**：`assertThat(x).isEqualTo(...)` / `extracting` / `hasSize` など流暢に。例外は `assertThatThrownBy`。
- **命名は振る舞いで**：「在庫不足なら例外を投げる」のように、何を保証するテストか日本語/英語で明示する。

## ハマりどころ / アンチパターン
- **`@SpringBootTest` の乱用で激遅（最頻）**：全Beanを起動するため1件でも重く、件数が増えるとCIが分単位で伸びる。**Controllerテストは `@WebMvcTest`、Repositoryテストは `@DataJpaTest`** にスライスする。`@SpringBootTest` は結合の要所だけ。
- **モック過剰でもろいテスト**：実装の内部呼び出し順まで `verify` で縛ると、リファクタのたびに赤くなる。**境界（外部API・時刻・乱数）だけモック**し、内部ロジックは実物で検証する。
- **テスト用DB/プロファイルの未分離**：本番DB設定がテストに漏れて破壊・遅延を招く。`@ActiveProfiles("test")` ＋ `src/test/resources` の設定で分離し、`ddl-auto: create-drop` で隔離する。
- **H2で通って本番で落ちる**：組込みDBの方言差を信用しすぎる。DB依存ロジックは Testcontainers で本物を当てる。
- **テスト間の状態リーク**：static変数・キャッシュ・コンテナの使い回しで順序依存に。スライスは各テストでコンテキストを分離、`@DataJpaTest` は自動ロールバックを活かす。
- **`@MockBean` 多用でコンテキスト再生成**：Bean構成が変わるたびにSpringがコンテキストをキャッシュし直し遅くなる。モック構成を揃えてキャッシュを効かせる。

## 関連
[junit5.md](./junit5.md) / [mockito.md](./mockito.md) / [assertj.md](./assertj.md) / [controller.md](./controller.md) / [repository.md](./repository.md) / [service.md](./service.md)
