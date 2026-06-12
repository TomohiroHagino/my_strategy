# Actuator（ヘルス/メトリクス）（Spring Boot 3）

## ひとことで言うと
**運用のための「点検口」エンドポイント群**。`/actuator/health`（生存確認）・`/actuator/metrics`（メトリクス）・`/actuator/info`（ビルド情報）などを追加するだけで生やせる、Spring Boot 標準の運用支援機能。

## 役割・なぜ必要か
- 本番で「アプリは生きているか」「DB/外部依存は繋がっているか」を**機械的に確認**するため。ロードバランサ・Kubernetes の liveness/readiness プローブがこれを叩く。
- メモリ・スレッド・HTTPレイテンシ・DBコネクションなどを**Micrometer 経由でメトリクス化**し、Prometheus → Grafana で可視化・アラートする。
- 自前で `/health` を書かずに、**標準化された運用面**を最小コストで手に入れられる。Boot 3 では Observability（メトリクス＋分散トレーシング）の中核。

## 基本の書き方（コード）
```yaml
# build.gradle / pom.xml に依存を追加
# implementation 'org.springframework.boot:spring-boot-starter-actuator'
# implementation 'io.micrometer:micrometer-registry-prometheus'  # Prometheus連携時
```
```yaml
# application.yml — 既定では /health と /info しか公開されない（残りは隠れている）
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus   # ★公開を明示的に列挙（* は本番で禁物）
      base-path: /actuator                        # 既定パス
  endpoint:
    health:
      show-details: when-authorized               # 詳細は認証済みのみ（never/when-authorized/always）
      probes:
        enabled: true                             # liveness/readiness を分離公開（K8s向け）
  metrics:
    tags:
      application: my-app                          # 全メトリクスに共通タグを付与
```
```java
// カスタムヘルスインジケータ：依存サービスの生死を health に組み込む
@Component
public class PaymentHealthIndicator implements HealthIndicator {
    private final PaymentClient client;
    public PaymentHealthIndicator(PaymentClient client) { this.client = client; }

    @Override
    public Health health() {
        try {
            client.ping();
            return Health.up().withDetail("gateway", "reachable").build();
        } catch (Exception e) {
            // DOWN を返すと /actuator/health 全体が DOWN になり 503 を返す
            return Health.down(e).build();
        }
    }
}
```
```java
// カスタムメトリクス：業務イベントを Micrometer で計測
@Service
public class OrderService {
    private final Counter orderCounter;
    public OrderService(MeterRegistry registry) {
        this.orderCounter = registry.counter("orders.created");
    }
    public void create(Order o) { /* ... */ orderCounter.increment(); }
}
```

## 実務での使い方・定番パターン
- **Kubernetes 連携**：`/actuator/health/liveness`（死んでたら再起動）と `/actuator/health/readiness`（準備できるまでトラフィック流さない）を probe に設定。起動直後にトラフィックを流して落とす事故を防ぐ。
- **Prometheus + Grafana**：`/actuator/prometheus` を Prometheus にスクレイプさせ、HTTPレイテンシ・JVMヒープ・GC・コネクションプールをダッシュボード化。SLO逸脱でアラート。
- **`/actuator/info` にビルド情報**：`spring-boot-gradle-plugin` の `buildInfo()` でコミットハッシュ・バージョンを埋め込み、稼働中のリビジョンを即確認できるようにする。
- **公開は必要最小限を列挙**：`include` は使うものだけ。`env`・`heapdump`・`threaddump`・`loggers` は機密性が高いので本番では原則出さない。
- **専用ポート分離**：`management.server.port` を business ポートと分けて内部ネットワークだけに公開し、外部から運用面を触れなくする。

## ハマりどころ / アンチパターン
- **エンドポイントを無防備に全公開（最重要・事故）**：`exposure.include: "*"` ＋認可なしだと、`/actuator/env` で**環境変数（＝DBパスワードやAPIキー）が丸見え**、`/actuator/heapdump` でヒープ流出、`/actuator/loggers` でログレベル改竄が可能。**必ず `include` を絞り、Spring Security で `/actuator/**` を認可**する。
  ```java
  // /actuator は管理者ロールのみ、health/info だけ匿名許可など
  http.authorizeHttpRequests(a -> a
      .requestMatchers("/actuator/health", "/actuator/info").permitAll()
      .requestMatchers("/actuator/**").hasRole("ADMIN"));
  ```
- **`show-details: always` を本番で**：未認証ユーザーに内部依存の障害詳細・接続先を晒す。`when-authorized` にする。
- **ヘルスチェックが重い**：`health` に外部API実呼び出しを大量に積むと、プローブのたびに負荷がかかりタイムアウトで誤検知。軽量化するかキャッシュする。
- **`HealthIndicator` の例外で意図せず全体 DOWN**：補助的な依存（任意のキャッシュ等）まで DOWN 判定にすると、本質的に生きているのに 503 を返す。重要度に応じて UP/UNKNOWN を返し分ける。
- **メトリクスの高カーディナリティ**：ユーザーIDやURLパスをタグに入れると時系列が爆発し Prometheus が膨張。タグは有限な値に限る。

## 関連
[security.md](./security.md) / [config_properties.md](./config_properties.md) / [async_scheduling.md](./async_scheduling.md)
