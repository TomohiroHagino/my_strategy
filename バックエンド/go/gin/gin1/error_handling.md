# エラー処理（err値 / Recovery）（Gin v1）

## ひとことで言うと
Go の **エラーは「例外」ではなく「値」**。関数は結果と一緒に `error` を返し、呼び出し側が `if err != nil` で**その場で判定**する。Gin では、その error を HTTP レスポンス（500 等）に変換し、想定外の **panic は Recovery ミドルウェアで回復**して 500 にする。

## 役割・なぜ必要か
- Go には try/catch が無い。エラーは戻り値なので、**握り潰しが目に見える**（無視すると変数が浮く）設計になっている。
- Web では「業務エラー（400/404）」と「想定外（500）」を**整形して返し、内部詳細は隠す**必要がある。
- 1ハンドラの panic でプロセス全体が落ちないよう、**Recovery で回復**してサービスを継続させる。

---

## 基本：エラーは値（if err != nil）
返ってきた error を**毎回**チェックし、ダメなら早期 return する。
```go
func getUser(c *gin.Context) {
    id := c.Param("id")
    user, err := repo.FindByID(c.Request.Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
            return // ← return を忘れると下の正常系も走る
        }
        // 想定外は 500（詳細はログに、ユーザーには出さない）
        log.Printf("getUser failed: %v", err)
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }
    c.JSON(http.StatusOK, user)
}
```

## Abort 系：以降のミドルウェアを止める
レスポンスを書いて**ハンドラチェーンを止める**には `Abort` 系を使う。`c.JSON` だけだと後続ミドルウェアは動き続ける。
```go
// JSON を書きつつ Abort（後続を止める）を一発で
c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
return

// ステータスだけ書いて止める
c.AbortWithStatus(http.StatusForbidden)
```

## c.Error：エラーをContextに溜める
`c.Error(err)` は**レスポンスを書かず**、Context のエラーリストに積むだけ。最後にミドルウェアでまとめて処理する集約パターンに使う。
```go
func handler(c *gin.Context) {
    if err := doWork(); err != nil {
        c.Error(err) // 溜めるだけ。レスポンスは後段のハンドラで決める
        c.AbortWithStatusJSON(http.StatusBadGateway, gin.H{"error": "upstream failed"})
        return
    }
}
```

## Recovery ミドルウェア（panic を回復）
`gin.Default()` は **Logger と Recovery** を最初から積む。Recovery は **panic を recover() して 500 を返し**、プロセス継続させる。
```go
r := gin.Default() // = gin.New() + Logger + Recovery

// 自前で組むなら New に明示的に積む
r2 := gin.New()
r2.Use(gin.Recovery())

// nil 参照などで panic しても Recovery が捕まえて 500 にする
func boom(c *gin.Context) {
    var p *int
    _ = *p // panic: nil pointer → Recovery が回復し 500
}
```
**panic は例外の代わりではない**。「ここに来たらバグ」という回復不能事態に限り、通常の業務エラーは `error` 値で返すのが原則。

## 集約エラーハンドリング（定番）
業務エラーに**型**を持たせ、1つのミドルウェアで HTTP ステータスへ変換すると、各ハンドラが薄くなる。
```go
type AppError struct {
    Code    int    // HTTP ステータス
    Message string // ユーザー向け文言
    Err     error  // 内部詳細（ログ用）
}
func (e *AppError) Error() string { return e.Message }

func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next() // ハンドラを先に走らせる
        if len(c.Errors) == 0 {
            return
        }
        last := c.Errors.Last().Err
        var ae *AppError
        if errors.As(last, &ae) {
            log.Printf("app error: %v", ae.Err) // 詳細はログだけ
            c.JSON(ae.Code, gin.H{"error": ae.Message})
            return
        }
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
    }
}
// ハンドラ側は c.Error(&AppError{Code: 404, Message: "not found", Err: err}) を積むだけ
```

## ハマりどころ / アンチパターン
- **`if err != nil` の徹底**：error を `_` で捨てるのは原則禁止。捨てるなら**理由をコメント**する。
- **panic を業務制御に使う**：panic はバグ通知用。「在庫不足」等は `error` で返す。Recovery 任せにしない。
- **Abort 忘れ**：`c.JSON` だけだと**後続ミドルウェアが動き続ける**。止めたいなら `AbortWithStatusJSON`。
- **return 忘れ**：エラー時に return しないと**正常系も実行**され二重書き込みになる。
- **本番でエラー詳細を露出**：`err.Error()` をそのまま返すとスタックや内部構造が漏れる。**詳細はログ、ユーザーには汎用文言**。
- **`errors.Is` / `errors.As` を使わない**：ラップされた error は `==` 比較で取れない。判定は `Is`/`As` を使う。

## 関連: [middleware.md](./middleware.md) / [binding_validation.md](./binding_validation.md)
