# トランザクション（@Transactional）（Spring Boot 3）

## ひとことで言うと
- **トランザクション= 複数のDB操作を「全部成功か、全部なかったことにする（ロールバック）」でくくる単位**。
- Spring では **`@Transactional` を付けたメソッドの開始〜終了を1つのトランザクション境界**にしてくれる。正常終了でコミット、例外で原則ロールバック。

## 役割・なぜ必要か
- 「口座Aから引き、口座Bへ足す」のように複数更新がある処理で、途中で失敗したら**中途半端な状態**（金が消える等）が残る。トランザクションが「全か無か（原子性）」を保証する。
- 手で `connection.commit()` / `rollback()` を書くと漏れ・例外時の取りこぼしが起きる。`@Transactional` は **AOP（プロキシ）でコミット／ロールバックを自動化**し、本処理を純粋に保つ。
- 境界を **Service 層に置く**のが定石。Controller は薄く、Repository は1操作単位。「1つの業務処理＝1トランザクション」を Service メソッドで表現する。

## 基本の書き方（コード）
### 基本（Service 層に付ける）
```java
import org.springframework.transaction.annotation.Transactional;
import org.springframework.stereotype.Service;

@Service
public class TransferService {
  private final AccountRepository repo;
  public TransferService(AccountRepository repo) { this.repo = repo; }

  @Transactional   // ★ このメソッド全体が1トランザクション
  public void transfer(Long fromId, Long toId, long amount) {
    var from = repo.findById(fromId).orElseThrow();
    var to   = repo.findById(toId).orElseThrow();
    from.withdraw(amount);     // どちらかで例外が起きたら
    to.deposit(amount);        // 両方の更新がロールバックされる
    repo.save(from);
    repo.save(to);
  }
}
```

### readOnly（参照のみ）
```java
@Transactional(readOnly = true)   // ★ 更新なしを明示 → フラッシュ抑制・最適化
public List<Account> list() {
  return repo.findAll();
}
```

### 伝播（propagation）
```java
import org.springframework.transaction.annotation.Propagation;

@Service
public class OrderService {

  @Transactional   // 既定 = Propagation.REQUIRED
  public void place(Order o) {
    save(o);
    audit.write(o);   // ↓ REQUIRES_NEW なら独立Txで、placeが落ちても監査は残る
  }
}

@Service
public class AuditService {
  // 親Txに参加せず常に新しいトランザクションで実行
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void write(Order o) { /* ... */ }
}
// 主な伝播:
//  REQUIRED      … 既存Txがあれば参加、なければ新規（既定。9割これ）
//  REQUIRES_NEW  … 常に新規Tx（親と独立。ログ/監査向き）
//  SUPPORTS      … あれば参加、なければTxなしで実行
//  MANDATORY     … 既存Tx必須（なければ例外）
//  NEVER / NOT_SUPPORTED … Tx内禁止 / 一時停止
```

### ロールバックの既定挙動（最重要）
```java
@Transactional
public void a() {
  repo.save(x);
  throw new IllegalStateException("boom");  // RuntimeException → ★ロールバックされる
}

@Transactional
public void b() throws IOException {
  repo.save(x);
  throw new IOException("io");  // checked例外 → ★既定ではロールバックされない（コミットされる）
}

// checked例外でもロールバックしたいなら明示
@Transactional(rollbackFor = Exception.class)
public void c() throws IOException { /* ... */ }
```

## 実務での使い方・定番パターン
- **境界は Service 層の public メソッド**に1つ。Controller / Repository には付けない。
- **読み取り専用は `readOnly = true`**。意図が伝わり、JPA のダーティチェック／フラッシュを抑えて軽くなる。
- **独立して残したい処理（監査ログ・通知記録）は `REQUIRES_NEW`**。本処理がロールバックしても消えない。
- **checked例外でロールバックさせたい時は `rollbackFor = Exception.class`** を明示（または独自例外を `RuntimeException` 継承にする）。
- **トランザクションは短く**。確定すべきものだけを包み、外部 API 呼び出しや重い計算は境界の外に出す。

## ハマりどころ / アンチパターン
- **自己呼び出しで効かない（最頻）**: 同一 Bean 内で `this.transfer()` と呼ぶと**プロキシを通らず** `@Transactional` が**完全に無視**される（AOP と同じ罠）。別 Bean に分ける／自身を注入して呼ぶ。
- **checked 例外でロールバックされない（重大）**: 既定は **unchecked（`RuntimeException`／`Error`）のみロールバック**。`IOException` 等で投げると**コミットされてしまう**。`rollbackFor` を明示。
- **例外を握りつぶす**: メソッド内で `try { } catch (Exception e) {}` して再送出しないと、Spring は例外を知れずコミットしてしまう。
- **`private`／`final` メソッドに付与**: プロキシが効かず無効。public な非 final メソッドに付ける。
- **トランザクション内で外部APIや長時間処理**: DB コネクションと行ロックを長く保持し、コネクション枯渇・デッドロックの原因。Tx 外へ。
- **`readOnly` なのに更新**: 一部DBで書き込みが弾かれたり、変更が反映されない事故。更新メソッドには付けない。
- **N+1 とフラッシュタイミング**: ループ内で都度 `save` せず、まとめて永続化／`saveAll` を検討。

## 関連
[service.md](./service.md) / [aop.md](./aop.md) / [repository.md](./repository.md)
