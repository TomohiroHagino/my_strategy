# 非同期 / スケジューリング（Spring Boot 3）

## ひとことで言うと
**「重い・遅い処理を裏のスレッドへ逃がす」仕組み**。`@Async` でメソッドを別スレッド実行し、`@Scheduled` で定期実行（cron/間隔）する。Boot 3.2＋Java 21 では**仮想スレッド**で大量の並行処理を低コストに捌ける。

## 役割・なぜ必要か
- メール送信・外部API呼び出し・集計など、**レスポンスから切り離してユーザーを待たせない**ために `@Async` を使う。
- バッチ集計・キャッシュ更新・定期クリーンアップなど、**時間トリガで自動実行**したい処理に `@Scheduled` を使う。
- これらは「キューに積むワーカー基盤（Sidekiq 等）」より軽量な**アプリ内スレッド完結**の選択肢。本格的なジョブ基盤が要るなら Spring Batch / メッセージング（Kafka・RabbitMQ）へ。
- スレッドプールを明示すれば**並行度・優先度・障害分離**を制御でき、業務影響を局所化できる。

## 基本の書き方（コード）
```java
// 起動クラス：機能を有効化（これが無いと @Async/@Scheduled は何も起きない）
@SpringBootApplication
@EnableAsync          // @Async を有効化
@EnableScheduling     // @Scheduled を有効化
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }
}
```
```java
@Service
public class MailService {
    // 戻り値 void：投げっぱなし（例外はログに出るが呼び出し側は気付けない）
    @Async
    public void sendWelcome(Long userId) {
        // 別スレッドで実行される。重い/遅い処理をここへ
    }

    // 戻り値 CompletableFuture：結果・例外を呼び出し側で受け取れる（推奨）
    @Async
    public CompletableFuture<Report> buildReport(Long id) {
        Report r = heavyAggregation(id);
        return CompletableFuture.completedFuture(r);
    }
}
```
```java
@Component
public class Jobs {
    // 固定レート：前回の「開始」から5秒ごと（処理時間に関係なく等間隔で起動を試みる）
    @Scheduled(fixedRate = 5000)
    public void poll() { /* ... */ }

    // 固定ディレイ：前回の「終了」から3秒後に次を起動（処理が重なりにくい）
    @Scheduled(fixedDelay = 3000)
    public void cleanup() { /* ... */ }

    // cron：毎日 02:30 に実行（秒 分 時 日 月 曜）
    @Scheduled(cron = "0 30 2 * * *", zone = "Asia/Tokyo")
    public void nightlyBatch() { /* ... */ }
}
```
```yaml
# application.yml — スレッドプール設定（未設定だと既定の貧弱なプールで詰まる）
spring:
  task:
    execution:        # @Async 用プール
      pool:
        core-size: 8
        max-size: 32
        queue-capacity: 100
      thread-name-prefix: async-
    scheduling:       # @Scheduled 用プール（既定は1スレッド＝直列）
      pool:
        size: 4
  threads:
    virtual:
      enabled: true   # ★ Boot 3.2＋Java 21：仮想スレッドを有効化（I/O待ちが多い処理に強い）
```

## 実務での使い方・定番パターン
- **戻り値は `CompletableFuture` で受ける**：`void` だと例外が握り潰される。結果合成は `thenCombine` / `allOf` で並行集約。
- **専用 `Executor` を定義**して `@Async("mailExecutor")` のように使い分け：メール送信とレポート生成でプールを分離し、片方の詰まりが全体を巻き込まないようにする。
```java
@Bean("mailExecutor")
public Executor mailExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(4); ex.setMaxPoolSize(8); ex.setQueueCapacity(50);
    ex.setThreadNamePrefix("mail-"); ex.initialize();
    return ex;
}
```
- **仮想スレッド（Java 21）**：DB・外部API待ちが支配的な処理は `spring.threads.virtual.enabled=true` で大量同時実行を低メモリに。CPUバウンドな処理には効かない点に注意。
- **`@Scheduled` は `fixedDelay` を基本**に：処理が重複しにくく、前段の遅延が次段を押し出さない。間隔は外部化（`@Scheduled(fixedDelayString = "${job.cleanup.delay}")`）して環境ごとに調整。
- **多重起動防止**：複数インスタンス（Pod）で同じ `@Scheduled` が同時に走ると二重処理になる。ShedLock 等で**分散ロック**を掛けるかリーダー選出する。

## これがあれば Redis/キューは要らない？（使い分け）
**結論：軽い処理なら `@Async`/`@Scheduled` で足り、Redisは不要。ただし「確実さ・分散・再起動で消えない」が要るなら別途キュー（Redis等）が要る。別物。**

```
 @Async / @Scheduled  … アプリ(JVM)の中で、自前スレッドで動かす（外部不要・お手軽）
 Redis＋キュー         … アプリの外に「ジョブの保管庫」を持つ（確実・分散・重装備）
```

| | `@Async`/`@Scheduled` | Redis等のキュー（Sidekiq的） |
|---|---|---|
| 動く場所 | アプリのメモリ内 | アプリの外（保管庫） |
| 再起動・クラッシュ | **ジョブが消える**（メモリ） | **残る**（積んである） |
| 複数台で分担 | 苦手 | 得意 |
| リトライ・失敗管理・可視化 | ほぼ無い | ある |
| 用途 | 軽い非同期・定期処理 | 重要・大量・確実が要るジョブ |

### ⚠️ 落とし穴：`@Scheduled` × 複数台 ＝ 重複実行
`@Scheduled` はアプリを**2台以上で動かすと全台で走る**（毎晩バッチが3台で3回）。
「1回だけ」にするには**台をまたぐ分散ロック**が要り、その鍵置き場として **Redis や DB（ShedLock 等）** を使う。
➡ つまり**スケールさせると逆にRedis/DBが要る**場面が出てくる（→ 実務節・ハマり節も参照）。

### ⚠️ そもそも Redis はジョブ専用じゃない
Redisを入れる理由はジョブ以外も多い：**キャッシュ / セッション共有 / 分散ロック / レート制限 / pub-sub**。
➡ だから「`@Async` があるからRedis不要」とはならず、**別の役割で残る**ことが多い。

### 目安
```
 軽い後処理(メール等)を待たせたくない          → @Async（Redis不要）
 1台運用で毎晩バッチ                            → @Scheduled（Redis不要）
 絶対失敗させたくない／大量／複数台で分担        → キュー(Redis/RabbitMQ/Kafka)
 複数台で定期実行を1回だけ                       → @Scheduled ＋ 分散ロック(Redis/DB)
 キャッシュ・セッション共有・レート制限がほしい  → Redis（ジョブと無関係に必要）
```
> **`@Async`/`@Scheduled`＝アプリ内のお手軽版、Redis/キュー＝外部の確実・分散版。置き換えではない。**

## ハマりどころ / アンチパターン
- **`@Async` の自己呼び出しは効かない（最頻）**：同じクラス内の `this.async()` 呼び出しはプロキシを通らず**同期実行**になる。別Beanに切り出して注入経由で呼ぶ。
  ```java
  // NG: 同一クラス内で this 経由 → 同期実行のまま
  public void run() { this.sendWelcome(id); }
  // OK: 別Bean(MailService)へ分離し、DIして呼ぶ
  ```
- **例外が静かに消える**：`void` の `@Async` で投げた例外は呼び出し側に伝わらない。`CompletableFuture` で受けるか `AsyncUncaughtExceptionHandler` を実装してログ＋通知する。
- **`@EnableAsync` / `@EnableScheduling` の付け忘れ**：アノテーションを書いても**ただの同期メソッド**として動く（無言で）。
- **スレッドプール未設定**：既定の `@Scheduled` プールは**1スレッド**＝ジョブが直列化し1本詰まると全部止まる。`@Async` 側も無設定だとタスク無制限生成で枯渇。必ずサイズを切る。
- **`@Async` 内で `@Transactional` が効かない誤解**：別スレッドにはトランザクション/セキュリティコンテキストが**引き継がれない**。境界を意識して設計する。
- **`fixedRate` で処理時間 > 間隔**：実行が積み上がりスレッドを食い潰す。重い処理は `fixedDelay` にする。

## 関連
[service.md](./service.md) / [aop.md](./aop.md) / [transactions.md](./transactions.md)
