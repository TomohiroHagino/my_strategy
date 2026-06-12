# Spring Security（認証・認可）（Spring Boot 3）

## ひとことで言うと
- **認証（Authentication）= 「あなたは誰か」** を確かめること（ログイン）。
- **認可（Authorization）= 「その操作をやってよいか」** を判断すること（権限）。
別物。Spring Security はこの2つを **サーブレットフィルタの連鎖（FilterChain）** として横断的に差し込むフレームワーク。

## 役割・なぜ必要か
- 認証だけでは「ログイン済みなら何でもできる」状態になり、**他人のリソースを操作できてしまう**（IDOR）。認可で「誰が・何を・どの操作を」許すか決める。
- Spring Security を入れると、依存追加だけで **全エンドポイントが既定で保護**される（何もしないと401/ログイン画面）。だから「どこを開けるか」を明示的に設計する発想になる。
- **Boot 3 / Security 6 の最重要変更**: 旧来の `WebSecurityConfigurerAdapter` を継承する設定スタイルは **廃止（削除）**。今は **`SecurityFilterChain` を Bean として返すコンポーネントベース設定** が唯一の正解。2系からの移行で最も引っかかる破壊的変更。

## 基本の書き方（コード）
### SecurityFilterChain Bean（フォームログイン）
```java
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SecurityConfig {

  // ★ Boot3/Security6: アダプタ継承ではなく Bean を返す
  @Bean
  SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/", "/login", "/css/**").permitAll() // 誰でも可
        .requestMatchers("/admin/**").hasRole("ADMIN")          // ADMINだけ
        .anyRequest().authenticated())                          // 残りは要ログイン
      .formLogin(form -> form.loginPage("/login").permitAll())
      .logout(logout -> logout.logoutSuccessUrl("/"));
    return http.build();
  }

  // ★ パスワードは必ずハッシュ化（BCrypt）。平文保存は禁止
  @Bean
  PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
  }
}
```

### UserDetailsService（DBからユーザーを引く）
```java
import org.springframework.security.core.userdetails.*;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

@Service
public class DbUserDetailsService implements UserDetailsService {
  private final UserRepository repo;
  public DbUserDetailsService(UserRepository repo) { this.repo = repo; }

  @Override
  public UserDetails loadUserByUsername(String username) {
    var u = repo.findByEmail(username)
        .orElseThrow(() -> new UsernameNotFoundException("not found: " + username));
    return User.withUsername(u.getEmail())
        .password(u.getPasswordHash())        // 既にBCryptハッシュ
        .authorities(new SimpleGrantedAuthority("ROLE_" + u.getRole()))
        .build();
  }
}
```

### メソッドセキュリティ（@PreAuthorize）
```java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@Configuration
@EnableMethodSecurity   // ★ これが無いと @PreAuthorize は効かない
public class MethodSecurityConfig {}

@Service
public class PostService {
  // 認可をメソッド境界で宣言。ロール／式言語(SpEL)で書ける
  @PreAuthorize("hasRole('ADMIN') or #post.ownerId == authentication.name")
  public void update(Post post) { /* ... */ }
}
```

### ステートレスAPI（JWT）の骨子
```java
@Bean
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
  http
    .securityMatcher("/api/**")
    .csrf(csrf -> csrf.disable())  // ★ トークン認証＝セッション不使用ならCSRF無効化
    .sessionManagement(sm -> sm.sessionCreationPolicy(
        org.springframework.security.config.http.SessionCreationPolicy.STATELESS))
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/auth/**").permitAll()
        .anyRequest().authenticated())
    .oauth2ResourceServer(o -> o.jwt(org.springframework.security.config.Customizer.withDefaults()));
  return http.build();
}
```

## 実務での使い方・定番パターン
- **フィルタチェーン分割**: 画面用（フォームログイン＋CSRF有効）と API 用（JWT＋ステートレス＋CSRF無効）を `securityMatcher` で別 `SecurityFilterChain` Bean に分ける。
- **認可は二段構え**: ①URL レベル（`authorizeHttpRequests`）でざっくり、②メソッドレベル（`@PreAuthorize`）で「自分の所有物だけ」など業務ルールを表現。**所有者チェックは必ずサーバ側で**。
- **`PasswordEncoder` は BCrypt**（または Argon2/`DelegatingPasswordEncoder`）。エンコーダ未登録だと `loadUserByUsername` のパスワード照合で例外。
- **JWT は Resource Server** として `oauth2ResourceServer().jwt()` を使い、署名検証・有効期限を委ねる。自前パースは避ける。
- **CORS** は `http.cors()` ＋ `CorsConfigurationSource` Bean で。フロント分離 SPA で必須。

## ハマりどころ / アンチパターン
- **`WebSecurityConfigurerAdapter` を使おうとする（最頻）**: Security 6 で削除済み。コンパイルが通らない／古い記事のコピペで詰まる。**必ず `SecurityFilterChain` Bean**へ。
- **全部 `permitAll()` にして穴を開ける**: デバッグ中に `anyRequest().permitAll()` と書いて戻し忘れ＝**全公開**。`anyRequest().authenticated()` を既定に。
- **ステートフルなのに CSRF を無効化**: フォーム／セッション認証で `csrf().disable()` すると CSRF 攻撃に無防備。**無効化はトークン認証のステートレスAPIだけ**。
- **`@EnableMethodSecurity` 付け忘れ**: `@PreAuthorize` が静かに無視され、認可ゼロで素通り。
- **ビューでボタンを隠しただけ＝認可ではない**: エンドポイントは直接叩ける。サーバ側で必ず判定。
- **`ROLE_` 接頭辞の不一致**: `hasRole("ADMIN")` は内部で `ROLE_ADMIN` を要求。権限文字列を `ROLE_` 付きで登録する（`hasAuthority` なら接頭辞なし）。

## 関連
[config_properties.md](./config_properties.md) / [exception_handling.md](./exception_handling.md) / [controller.md](./controller.md)
