# AOP（横断的関心事）（Spring Boot 3）

## ひとことで言うと
- **AOP（Aspect Oriented Programming）= 本処理と直接関係ないが多くの場所で必要な処理（ロギング・監査・トランザクション・キャッシュ等）を、本処理から切り出してまとめる仕組み**。
- これら「あちこちに散らばる関心事」を **横断的関心事（cross-cutting concern）** と呼ぶ。

## 役割・なぜ必要か
- ログ出力・実行時間計測・権限チェック・トランザクション境界などを各メソッドに直書きすると、**本来のロジックがノイズに埋もれ、変更も全箇所に波及**する（DRY違反）。
- AOP は「どこに（ポイントカット）」「何を（アドバイス）」を1か所に宣言し、対象メソッドの**前後に自動で差し込む**。本処理は純粋なまま保てる。
- **Spring の `@Transactional` `@Cacheable` `@Async` `@PreAuthorize` 自体が AOP で実現**されている。つまり AOP を理解すると Spring の魔法の正体が分かる。
- Spring の AOP は **プロキシベース**: 対象 Bean を包む代理オブジェクト（プロキシ）を DI コンテナが差し込み、呼び出しを横取りしてアドバイスを実行する。

## 基本の書き方（コード）
### 依存と Aspect 定義
```java
// build.gradle: implementation 'org.springframework.boot:spring-boot-starter-aop'
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect       // ★ これが横断ロジックの入れ物
@Component    // ★ Bean として登録されないと有効化されない
public class LoggingAspect {

  // ポイントカット: service パッケージ配下の全 public メソッドが対象
  @Pointcut("execution(* com.example.app.service..*(..))")
  void serviceMethods() {}

  // 前: メソッド実行前に走る
  @Before("serviceMethods()")
  public void logBefore(org.aspectj.lang.JoinPoint jp) {
    System.out.println("CALL " + jp.getSignature());
  }

  // 周辺: 前後を包む。proceed() で本処理を呼ぶ（最も柔軟）
  @Around("serviceMethods()")
  public Object measure(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.nanoTime();
    try {
      return pjp.proceed();             // ★ ここで本来のメソッドが動く
    } finally {
      long ms = (System.nanoTime() - start) / 1_000_000;
      System.out.println(pjp.getSignature() + " took " + ms + "ms");
    }
  }
}
```

### アノテーション駆動のカスタム監査
```java
import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Audited {}   // 付けたメソッドだけ監査したい

@Aspect
@Component
public class AuditAspect {
  // @Audited が付いたメソッドだけにマッチ
  @AfterReturning(pointcut = "@annotation(com.example.app.Audited)", returning = "result")
  public void record(org.aspectj.lang.JoinPoint jp, Object result) {
    System.out.println("AUDIT " + jp.getSignature() + " -> " + result);
  }
}

@Service
public class OrderService {
  @Audited
  public Order place(OrderRequest req) { /* ... */ return new Order(); }
}
```

### アドバイスの種類
```java
// @Before        : 前に実行
// @AfterReturning: 正常終了後（戻り値を受け取れる）
// @AfterThrowing : 例外送出時（例外を受け取れる）
// @After         : finally 相当（成否問わず）
// @Around        : 前後を包む。proceed() 呼び出しと戻り値/例外を制御できる最強
```

## 実務での使い方・定番パターン
- **ロギング／実行時間計測**: `@Around` で全 Service を計測。遅いメソッドの特定に有効。
- **監査ログ**: `@Audited` のようなカスタムアノテーション＋ `@AfterReturning`／`@AfterThrowing` で「誰が・何を・結果」を記録。
- **フレームワーク機能はそのまま使う**: トランザクションは自作せず `@Transactional`、キャッシュは `@Cacheable` を使う（いずれも AOP 実装）。自作 Aspect は本当に横断的なものだけに絞る（YAGNI）。
- **ポイントカットは定数化**: `@Pointcut` で名前を付けて使い回し、式の重複（コピペ）を避ける。
- **順序制御**: 複数 Aspect が重なる時は `@Order` で優先度を明示。

## ハマりどころ / アンチパターン
- **自己呼び出しで AOP が効かない（最頻・最重大）**: 同一 Bean 内で `this.other()` と呼ぶと**プロキシを通らず**、`@Around`／`@Transactional`／`@Cacheable` が**全て無視**される。原因が見えず数時間溶かす定番。回避＝別 Bean に分離する／自分自身を注入して `self.other()` 経由で呼ぶ。
- **`@Aspect` に `@Component` 付け忘れ**: Bean 登録されず、Aspect が静かに無効。
- **ポイントカット式の誤り**: `execution(...)` のパッケージ／戻り値／引数指定をミスすると、マッチせず無反応（エラーも出ない）。まず広めに当てて段階的に絞る。
- **`final` メソッド／クラス**: CGLIB プロキシは継承で作るため、`final` だとプロキシ化できずアドバイスが当たらない。
- **`@Around` で `proceed()` 呼び忘れ**: 本処理が実行されず、戻り値が `null` になる事故。
- **重い処理を毎メソッドに**: アドバイス内で同期 I/O やリモート呼び出しをすると全 Service が遅くなる。軽量に保つ。

## 関連
[transactions.md](./transactions.md) / [service.md](./service.md) / [security.md](./security.md)
