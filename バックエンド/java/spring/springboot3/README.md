# Spring Boot 3 実務リファレンス（索引）

> **この版 = Spring Boot 3 系（Spring Framework 6 / Java 17+、21対応）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。

## この版のポイント（Spring Boot 3 で何が変わったか）
- **Jakarta EE への移行（最重要）**: パッケージが **`javax.*` → `jakarta.*`** に全面変更。2系からの移行で一番引っかかる破壊的変更（`jakarta.persistence` `jakarta.servlet` 等）。
- **Java 17 がベースライン**（21もサポート）。
- **GraalVM native image を公式サポート**（AOTコンパイルで起動爆速・省メモリ）。
- **Observability 刷新**: Micrometer ベースのメトリクス＋分散トレーシング統合。
- **3.1**: Testcontainers / Docker Compose 連携（`spring-boot-docker-compose`）。
- **3.2**: **仮想スレッド（Java 21）** サポート、**`RestClient`**・`JdbcClient` 導入。

## レイヤード構成（Springの基本形）
```
 HTTPリクエスト
   │
   ▼
 Controller（受付：URL↔メソッド、入出力の変換）   @RestController
   │  呼ぶ
   ▼
 Service（業務ロジック・トランザクション）          @Service
   │  呼ぶ
   ▼
 Repository（DBアクセス）                           @Repository / JpaRepository
   │
   ▼
 Entity ←→ DB（テーブル）                            @Entity（JPA/Hibernate）
```
> どの層も **DIコンテナがインスタンスを生成して結びつける**（自分で new しない）のが土台。

## 他FW（Rails / Django / Laravel）との用語対応
Springは「Model」「View」という単一ファイルを持たない。**役割ごとに分割**されているため、対応はこう:

| 他FWの概念 | Springでの担当 |
|---|---|
| **Model**（Fat Model） | Entity（データ）＋ Repository（DB）＋ Service（ロジック）に**3分割** → [model.md](./model.md) が地図 |
| **View** | REST=JSON（DTO+Jackson）/ サーバHTML=Thymeleaf → [view.md](./view.md) |
| Controller | [controller.md](./controller.md)（ほぼ同じ） |
| ORM/クエリ | [entity_jpa.md](./entity_jpa.md) / [repository.md](./repository.md) |

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（Spring Initializr / 起動）
- [project_structure.md](./project_structure.md) … レイヤード構成（Controller/Service/Repository）とは
- [model.md](./model.md) … モデルとは（Springでの“モデル”の地図＝Entity/Repository/Service への分割）

### 核（DIコンテナ）
- [bean.md](./bean.md) … Bean化とは（平易な入口：クラスをSpringに管理させること）
- [ioc_di.md](./ioc_di.md) … IoC / DI / Bean とは（Spring最大の肝・詳しい版）
- [beans_config.md](./beans_config.md) … Bean定義 / @Configuration / 自動設定とは

### 3層
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層が何を返すか（全体を1枚で俯瞰）
- [controller.md](./controller.md) … コントローラ（@RestController）とは
- [service.md](./service.md) … サービス層とは
- [repository.md](./repository.md) … リポジトリ（Spring Data JPA）とは
- [entity_jpa.md](./entity_jpa.md) … エンティティ / JPA / Hibernate / N+1 とは
- [view.md](./view.md) … ビューとは（REST=JSON / サーバHTML=Thymeleaf）

### リクエスト処理
- [dto.md](./dto.md) … DTO（Entityと分ける理由）とは
- [entity_or_dto.md](./entity_or_dto.md) … Modelの箱に入れるのは Entity か DTO か（つまずき解説）
- [validation.md](./validation.md) … バリデーション（Bean Validation）とは
- [exception_handling.md](./exception_handling.md) … 例外ハンドリング（@ControllerAdvice）とは
- [config_properties.md](./config_properties.md) … 設定（application.yml / プロファイル）とは

### 横断・運用
- [security.md](./security.md) … Spring Security（認証・認可）とは
- [aop.md](./aop.md) … AOP（横断的関心事）とは
- [transactions.md](./transactions.md) … トランザクション（@Transactional）とは
- [async_scheduling.md](./async_scheduling.md) … 非同期 / スケジューリングとは
- [actuator.md](./actuator.md) … Actuator（ヘルス/メトリクス）とは

### テスト・罠
- [testing.md](./testing.md) … テスト（JUnit5 / MockMvc / Testcontainers）とは
- [junit5.md](./junit5.md) … JUnit 5（テスト実行基盤：@Test / @ParameterizedTest / @Nested）
- [mockito.md](./mockito.md) … Mockito（依存のモック化：@Mock / when / verify / ArgumentCaptor）
- [assertj.md](./assertj.md) … AssertJ（流暢なアサーション：assertThat / assertThatThrownBy）
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（早見表）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Spring Boot 3）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
