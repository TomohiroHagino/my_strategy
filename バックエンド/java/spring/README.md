# Spring / Spring Boot

## 一言で
Javaで圧倒的定番のアプリケーションフレームワーク。**DI（依存性注入）コンテナ**を核に、Web・データ・セキュリティ・バッチまでを包括。**Spring Boot** が「設定地獄」を解消し、今はBoot前提が主流。

## 特徴
- **IoC / DI コンテナが核**: オブジェクト（Bean）の生成と結びつけをSpringが管理。`@Autowired`/コンストラクタ注入。
- **アノテーション駆動**: `@RestController` `@Service` `@Repository` `@Entity` などで役割を宣言。
- **レイヤード構成**: **Controller（受付）→ Service（業務ロジック）→ Repository（DB）** の3層が定番。
- **Spring Boot**: 自動設定（auto-configuration）・組込サーバ（Tomcat内蔵）・スターター依存で、即起動できる。
- **Spring Data JPA / Spring Security / Actuator** など統合が巨大。

## どういう使い方をするのか
- **エンタープライズWeb / REST API**。
- **マイクロサービス**（Spring Boot / Spring Cloud）。
- **バッチ処理**（Spring Batch）。

## 強み
- 成熟・安定・求人が多い。自動設定で立ち上げが速い（Boot）。
- セキュリティ・トランザクション・データアクセスが堅実。
- エコシステムと実績が圧倒的。

## 弱み・注意点
- 抽象・自動設定が多く「魔法」の理解コストが高い（何が起きているか追いにくい）。
- アノテーションの暗黙挙動・設定の罠。
- 起動・メモリが重め（GraalVM native image で改善）。

## エコシステム・周辺ツール
- Spring Boot / Spring Data / Spring Security / Spring Batch / Spring Cloud
- ビルド: Maven / Gradle、ORM: Hibernate（JPA）
- 運用: **Actuator**（ヘルス/メトリクス）、Micrometer（観測）
- テスト: JUnit 5 / Mockito / **Testcontainers**

## ひとことメモ（自分の実感）
- （現役として触った所感を後から追記）

## このフォルダの構成
- [springboot3/](./springboot3/) … **Spring Boot 3 実務リファレンス（フラッグシップ）**。始め方〜DI〜3層〜JPA〜Security〜テスト〜罠まで、項目=1ファイル。
- （Spring Boot 3 = Spring Framework 6 / Java 17+。差分は springboot3 の「この版のポイント」に補記。他バージョンは一旦作らない）
