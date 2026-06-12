# ミドルウェア（Middleware）（Gin v1）

## ひとことで言うと
ハンドラの前後に挟み込む共通処理。実体は **`func(c *gin.Context)`** という、ハンドラと同じ型の関数。`r.Use(...)` で登録すると、リクエストごとに連鎖を通って実行される。

## 役割・なぜ必要か
- 認証・ログ・CORS・リカバリ・計測など「**全ルート（または一部）に共通して効かせたい処理**」を、各ハンドラに書かずに一箇所へ寄せる仕組み。
- Gin は登録されたミドルウェアを**チェーン（数珠つなぎ）**にして順番に呼ぶ。各ミドルウェアは `c.Next()` を呼ぶことで「次の処理（次のミドルウェア or 最終ハンドラ）」へ制御を渡す。
- `c.Next()` の**前**に書いた処理は「前処理」、**後**に書いた処理は「後処理」になる。これでログの開始/終了やレイテンシ計測が自然に書ける。
- 途中で `c.Abort()` を呼べば**以降の処理を止められる**ので、認証失敗やレート超過で早期リターンできる。

## 基本の書き方（コード）
```go
// 計測ミドルウェア：c.Next() の前後で処理を挟む
func Timing() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()      // ── 前処理（リクエスト到着時）
        c.Next()                 // ── 次（後続ミドルウェア→ハンドラ）へ
        latency := time.Since(start) // ── 後処理（応答が決まった後）
        log.Printf("%s %s -> %d (%v)",
            c.Request.Method, c.Request.URL.Path, c.Writer.Status(), latency)
    }
}

// 認証ミドルウェア：失敗したら c.Abort() で中断する
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            // JSON を返したうえで Abort。return も忘れない
            c.AbortWithStatusJSON(http.StatusUnauthorized,
                gin.H{"error": "認証が必要です"})
            return
        }
        c.Set("userID", 42) // 後続へ値を渡す（context.md 参照）
        c.Next()            // 認証OK → 後続へ
    }
}

func main() {
    r := gin.New()             // 何も入っていない素の Engine
    r.Use(gin.Recovery())      // 組込：panic を 500 に変換して回復
    r.Use(Timing())            // 全ルートに計測を効かせる

    // グループ単位でミドルウェアを足す
    api := r.Group("/api", AuthRequired())
    api.GET("/me", func(c *gin.Context) {
        uid, _ := c.Get("userID")
        c.JSON(http.StatusOK, gin.H{"userID": uid})
    })

    // ルート単位（第2引数以降にいくつでも並べられる）
    r.GET("/health", Timing(), func(c *gin.Context) {
        c.String(http.StatusOK, "ok")
    })

    r.Run(":8080")
}
```

## 実務での使い方・定番パターン
- **組込みを使う**：`gin.Logger()`（アクセスログ）と `gin.Recovery()`（panic回復）。`gin.Default()` はこの2つを最初から `Use` 済み。自前で制御したいときは `gin.New()` から組み立てる。
- **適用範囲を3段で使い分け**：
  - 全体 → `r.Use(...)`
  - グループ → `r.Group("/api", Mw1, Mw2)` または `g.Use(...)`
  - 単一ルート → `r.GET("/x", Mw, handler)`
- **認証**：トークン検証 → OKなら `c.Set("userID", ...)` でハンドラに渡し `c.Next()`、NGなら `c.AbortWithStatusJSON(...)` + `return`。
- **CORS**：プリフライト（`OPTIONS`）はヘッダを付けて `c.AbortWithStatus(204)` で即返す。本番では `github.com/gin-contrib/cors` を使うのが安全。
- **ログ**：`c.Next()` をはさんでレイテンシとステータスを記録。`c.Writer.Status()` は応答確定後に正しい値になる。
- **設定の渡し方**：`func AuthRequired(secret string) gin.HandlerFunc { return func(c *gin.Context){...} }` のように**クロージャで設定を注入**するのが定番。
- **エラー集約**：ハンドラで `c.Error(err)` を積んでおき、終端ミドルウェアで `c.Errors` をまとめて応答する設計もよく使う。→ [error_handling.md](./error_handling.md)

```go
// CORS の最小例（本番は gin-contrib/cors 推奨）
func CORS() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Authorization,Content-Type")
        if c.Request.Method == http.MethodOptions {
            c.AbortWithStatus(http.StatusNoContent) // プリフライトは即終了
            return
        }
        c.Next()
    }
}
```

## ハマりどころ / アンチパターン
- **`c.Abort()` を呼ばないと処理が続行する**：`c.JSON(401, ...)` だけ書いて `Abort` しないと、後続ミドルウェアと**ハンドラがそのまま実行**され、二重に書き込もうとして崩れる。中断したいときは必ず `Abort`（または `AbortWithStatusJSON`）＋ `return`。
- **`Abort` してもその関数は止まらない**：`Abort` は「**後続**を呼ばない」フラグを立てるだけ。**自分の関数内**は `return` しない限り下まで流れる。`Abort` の直後に `return` を書くこと。
- **`c.Next()` の前後の取り違え**：計測やログの「終わった後」の処理を `c.Next()` の**前**に書くと、まだ処理前の値（ステータス0など）を拾ってしまう。後処理は必ず `c.Next()` の**後**へ。
- **登録順が効く**：`Recovery` は早めに `Use` しないと、それより後ろのミドルウェアでの panic を拾えない。ログ・認証・CORS なども順番で挙動が変わる。
- **`gin.Default()` に Recovery/Logger を重ねて二重登録**：`Default()` は既に入っている。自前で足すなら `gin.New()` から。
- **重い同期処理をミドルウェアに置く**：全リクエストに効くので、DBアクセス等を無条件に挟むと全体が遅くなる。必要なルートだけに絞る。

## 関連: [context.md](./context.md) / [error_handling.md](./error_handling.md)
