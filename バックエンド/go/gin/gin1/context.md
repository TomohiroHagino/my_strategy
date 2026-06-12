# `gin.Context`（`c`）（Gin v1）

## ひとことで言うと
1リクエストの**入力・出力・途中の状態**をすべて持ち運ぶ中心オブジェクト。ハンドラもミドルウェアも `func(c *gin.Context)` でこれを受け取り、`c.JSON` 等で応答する。

## 役割・なぜ必要か
- Gin の世界では「リクエストから値を取る」「レスポンスを書く」「ミドルウェア間で値を共有する」という**主要操作がすべて `c` に集約**されている。標準の `http.ResponseWriter` / `*http.Request` を薄くラップし、便利メソッドを足したもの。
- **リクエストごとに1つ生成され、そのリクエストの寿命の間だけ有効**。チェーン（middleware→handler）を貫いて同じ `c` が渡るので、前段で詰めた情報を後段で読める。
- 内部にプール（再利用）の仕組みがあるため、**リクエストの処理が終わった `c` を後生で（goroutine等で）使い回してはいけない**。

## 基本の書き方（コード）
```go
func handler(c *gin.Context) {
    // ── 入力を取る ───────────────────────────────
    id := c.Param("id")                 // パス :id
    q := c.Query("q")                   // クエリ ?q=...
    name := c.PostForm("name")          // フォーム body
    auth := c.GetHeader("Authorization")// ヘッダ
    _ = c.Request                       // 生の *http.Request も触れる

    // ── ミドルウェアが詰めた値を取り出す ──────────
    v, exists := c.Get("userID")        // (any, bool)
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "未認証"})
        return
    }
    userID, ok := v.(int)               // 型アサーションが必要
    if !ok {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "userID型不正"})
        return
    }

    // ── 出力する（応答は1リクエストにつき1回） ────
    c.JSON(http.StatusOK, gin.H{
        "id": id, "q": q, "name": name, "auth": auth, "userID": userID,
    })
}

// ミドルウェア側：c.Set で後段へ値を渡す
func setUser() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Set("userID", 7)   // キーは string、値は any
        c.Next()
    }
}
```

```go
// goroutine へ渡すときは必ず c.Copy()
func asyncHandler(c *gin.Context) {
    cc := c.Copy() // 読み取り専用の安全なコピー
    go func() {
        time.Sleep(2 * time.Second)
        // cc は使ってよい。元の c はこの時点で無効かもしれない
        log.Printf("path=%s", cc.Request.URL.Path)
    }()
    c.JSON(http.StatusOK, gin.H{"status": "受付けました"})
}
```

## 実務での使い方・定番パターン
- **入力メソッドの使い分け**：パス=`c.Param`、クエリ=`c.Query`/`c.DefaultQuery`、フォーム=`c.PostForm`、JSONボディ=`c.ShouldBindJSON`、ヘッダ=`c.GetHeader`。→ [request_response.md](./request_response.md)
- **`c.Set` / `c.Get` で前段→後段に値を渡す**：認証ミドルウェアが `c.Set("userID", id)` し、ハンドラが `c.Get("userID")` で受ける、が王道。→ [middleware.md](./middleware.md)
- **型付きゲッター**：`c.GetString("k")` / `c.GetInt("k")` / `c.GetBool("k")` など、よく使う型は専用メソッドがあり、型アサーションを省ける（無ければゼロ値）。`c.MustGet("k")` は無ければ panic するので、確実に存在する前提のときだけ使う。
- **出力は1回**：`c.JSON` / `c.String` / `c.Status` のいずれかで1回だけ書く。複数回書くと `http: superfluous WriteHeader` 等になる。
- **`c.Request` / `c.Writer`**：生の `*http.Request` と `gin.ResponseWriter`（`http.ResponseWriter` 互換）。`c.Writer.Status()` で確定後のステータスを取れる。
- **タイムアウト連携**：`c.Request.Context()` を DB クライアント等へ渡すと、クライアント切断で処理をキャンセルできる。

## ハマりどころ / アンチパターン
- **`c.Get` の型アサーション忘れ**：`c.Get` は `(any, bool)` を返す。`v.(int)` のアサーションが要り、失敗すると panic する。`v, ok := raw.(int)` の2値で受けて `ok` を確認するか、型付きゲッター（`c.GetInt`）を使う。
- **goroutine に `c` を直接渡す**：`c` はリクエスト終了後にプールへ戻り再利用される。別 goroutine で生の `c` を触ると**別リクエストのデータと混線**したり panic する。**`cc := c.Copy()` を渡す**こと。
- **`c.Copy()` で書き込もうとする**：コピーは読み取り目的。コピーへ `c.JSON` しても元の応答には反映されない。
- **`c` をフィールドに保存して使い回す**：構造体に `c` を持たせて後で使う等は寿命違反。必要な値だけ取り出して渡す。
- **`c.MustGet` の多用**：キーが無いと panic。ミドルウェアが必ず `Set` する保証が無い場所では `c.Get` + `exists` チェックを。
- **リクエストスコープの誤解**：`c.Set` した値はそのリクエスト内だけ。グローバル状態の代わりにはならない。

## 関連: [middleware.md](./middleware.md) / [request_response.md](./request_response.md)
