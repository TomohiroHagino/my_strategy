# 認証・認可（Authentication / Authorization）（Gin v1）

## ひとことで言うと
- **認証（Authentication）= 「あなたは誰か」** を確かめること（ログイン）。
- **認可（Authorization）= 「その操作をやってよいか」** を判断すること（権限）。
Gin 自体は認証機構を内蔵しない。**ミドルウェアでトークン/セッションを検証し、`c.Set` でユーザを後続へ渡す**のが基本形。

## 役割・なぜ必要か
- 認証だけでは「ログイン済みなら何でもできる」状態になり、**他人のリソースを操作できてしまう**（IDOR）。
- 認可は「このユーザーが、この対象に、この操作を許されているか」をハンドラ/ミドルウェアの1か所に集約する。
- Gin は薄いので、**JWT 検証・パスワードハッシュ・権限判定はライブラリ＋自前**で組む。素朴に書くと検証漏れが起きやすい。

## 基本の書き方（コード）
### パスワードハッシュ（bcrypt）
```go
import "golang.org/x/crypto/bcrypt"

// 登録時：平文を保存せずハッシュ化して保存
hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
if err != nil { /* 500 */ }
// 検証時：保存ハッシュと平文を比較（一致で nil）
if err := bcrypt.CompareHashAndPassword(stored, []byte(password)); err != nil {
    c.JSON(401, gin.H{"error": "認証失敗"}); return
}
```

### JWT 発行（golang-jwt/jwt）
```go
import "github.com/golang-jwt/jwt/v5"

claims := jwt.MapClaims{
    "sub": user.ID,
    "exp": time.Now().Add(2 * time.Hour).Unix(), // 期限は必須
}
token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
signed, err := token.SignedString([]byte(os.Getenv("JWT_SECRET"))) // secretはenvから
```

### JWT 検証ミドルウェア → c.Set で渡す
```go
func AuthRequired() gin.HandlerFunc {
    secret := []byte(os.Getenv("JWT_SECRET"))
    return func(c *gin.Context) {
        raw := strings.TrimPrefix(c.GetHeader("Authorization"), "Bearer ")
        tok, err := jwt.Parse(raw, func(t *jwt.Token) (any, error) {
            // 署名アルゴリズムを必ず検証（alg=none 攻撃対策）
            if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, errors.New("unexpected alg")
            }
            return secret, nil
        })
        if err != nil || !tok.Valid {
            c.AbortWithStatusJSON(401, gin.H{"error": "無効なトークン"}); return
        }
        claims := tok.Claims.(jwt.MapClaims)
        c.Set("userID", claims["sub"]) // 後続ハンドラは c.MustGet("userID")
        c.Next()
    }
}
```

### 保護ルートにミドルウェアを適用
```go
auth := r.Group("/api", AuthRequired())   // グループ単位で適用
auth.GET("/me", meHandler)
auth.PUT("/posts/:id", updatePostHandler)
```

### セッション方式（gin-contrib/sessions）
```go
import ("github.com/gin-contrib/sessions"; "github.com/gin-contrib/sessions/cookie")

store := cookie.NewStore([]byte(os.Getenv("SESSION_SECRET")))
r.Use(sessions.Sessions("mysession", store))
// ログイン時
s := sessions.Default(c); s.Set("userID", user.ID); s.Save()
// 検証ミドルウェア内
if sessions.Default(c).Get("userID") == nil { c.AbortWithStatus(401); return }
```

## 実務での使い方・定番パターン
- **認可の第一歩はスコープ**：`db.Where("user_id = ?", c.MustGet("userID")).First(&post, id)` で他人のレコードを引かせない。
- **権限チェックはハンドラ先頭で明示**：`if post.UserID != userID && !isAdmin { c.AbortWithStatus(403); return }`。
- **APIはJWT（ステートレス）、Web画面はセッション（Cookie）** と用途で使い分ける。
- secret は `os.Getenv` / viper 経由で注入（→ [config_env.md](./config_env.md)）。コードに直書きしない。
- 検証ロジックは**ミドルウェア1か所**に集約し、各ハンドラは `c.MustGet` で受け取るだけにする（→ [middleware.md](./middleware.md)）。

## ハマりどころ / アンチパターン
- **認証だけして認可を忘れる（最頻・最重大）**：ログインさえ通れば `db.First(&post, id)` で他人の投稿を編集・削除できる。必ず `user_id` スコープ or 権限チェックを入れる。
- **本人/権限チェック漏れ**：`PUT /posts/:id` で「自分の投稿か」を確認しないと改ざんされる。所有者判定を必ず書く。
- **secret のハードコード/コミット**：`[]byte("mysecret")` 直書きは漏洩源。env管理し、漏れたら再生成。
- **署名アルゴリズム未検証**：`jwt.Parse` のコールバックで `*jwt.SigningMethodHMAC` を確認しないと `alg=none` 偽装を許す。
- **`exp`（期限）を入れない / 検証しない**：無期限トークンは盗まれたら永続的に有効。発行時に必ず `exp` を付ける。
- **`tok.Valid` の確認漏れ**：`err == nil` だけで通すと不正トークンを許す場合がある。`tok.Valid` も必ず見る。
- **平文パスワード保存/ログ出力**：必ず bcrypt でハッシュ化。リクエストログに password を残さない。

## 関連
[middleware.md](./middleware.md) / [config_env.md](./config_env.md)
