# ルーティング（Gin v1）

## ひとことで言うと
「どの URL・どの HTTP メソッドのとき、どの関数を呼ぶか」の対応付け。**`r.GET("/path", handler)` のようにメソッド別に登録**し、パスの一部を変数（パラメータ）として取り出せる。

## 役割・なぜ必要か
- HTTP リクエストは「メソッド（GET/POST…）＋パス（/users/3）」で来る。これを処理関数に振り分けるのがルーティング。
- Gin のルータは内部で **radix tree（基数木）** を使い、ルートが増えても高速にマッチする。
- **パスパラメータ**（`/users/:id` の `id`）や**クエリ**（`?q=foo`）で、可変な入力を受け取れる。
- **ルートグループ**で共通プレフィックス（`/api`）や共通ミドルウェア（認証など）をまとめて、定義を整理できる。

## 基本の書き方（コード）
```go
r := gin.Default()

// メソッド別に登録
r.GET("/users/:id", getUser)       // 取得
r.POST("/users", createUser)       // 作成
r.PUT("/users/:id", updateUser)    // 更新（全置換）
r.DELETE("/users/:id", deleteUser) // 削除

func getUser(c *gin.Context) {
	// パスパラメータ :id → c.Param("id")（必ず文字列）
	id := c.Param("id")

	// クエリ ?q=foo → c.Query（無ければ ""）。既定値付きは DefaultQuery
	q := c.Query("q")
	page := c.DefaultQuery("page", "1")

	c.JSON(200, gin.H{"id": id, "q": q, "page": page})
}
```

```go
// ワイルドカード *param は「残りのパス全部」を1つに取り込む（/ を含む）
// /files/img/a.png → c.Param("filepath") == "/img/a.png"
r.GET("/files/*filepath", func(c *gin.Context) {
	c.JSON(200, gin.H{"path": c.Param("filepath")})
})
```

```go
// ルートグループ：共通プレフィックス + グループ単位のミドルウェア
api := r.Group("/api")
{
	v1 := api.Group("/v1")
	{
		v1.GET("/health", health)                 // GET /api/v1/health
	}

	// グループにだけ認証を効かせる
	auth := api.Group("/admin", AuthRequired())   // ミドルウェアを第2引数に
	{
		auth.GET("/stats", stats)                  // 認証必須
	}
}
```

## 実務での使い方・定番パターン
- **REST の素直なマッピング**：`GET /users`（一覧）`POST /users`（作成）`GET /users/:id` `PUT/PATCH /users/:id` `DELETE /users/:id`。
- **`/api/v1` のバージョングループ**：破壊的変更時に `/api/v2` を並走させられる。
- **ミドルウェアの階層化**：全体は `r.Use(...)`、特定範囲はグループ第2引数や `group.Use(...)` で。→ [middleware.md](./middleware.md)
- **パラメータは必ず検証・変換**：`c.Param("id")` は文字列なので、数値なら `strconv.Atoi` で変換し、失敗時は 400 を返す。
- **`r.Run` の起動エラーは処理**：`if err := r.Run(":8080"); err != nil { log.Fatal(err) }`。本番では `http.Server` を自前で組んでグレースフルシャットダウンする構成も多い。

## ハマりどころ / アンチパターン
- **`:id` と静的パスの競合**：同じ階層に `/users/:id` と `/users/new` を両方置くと、Gin は「同じ位置にワイルドカードと静的パスは共存不可」とパニックする（起動時）。設計で `/users/new` を別階層にするか、ID側で分岐する。
- **`*action`（キャッチオール）は末尾にしか置けない**：`/files/*filepath/info` のような後続セグメントは定義できない。
- **`c.Param` と `c.Query` の取り違え**：`:id` はパス、`?id=` はクエリ。別物。
- **末尾スラッシュの差**：既定では `/users` と `/users/` はリダイレクトで吸収される（`RedirectTrailingSlash`）。挙動を固定したいなら設定を確認する。
- **グループのミドルウェアが効かない**：`r.Group("/api")` の戻り値（子ルータ）にルートを登録していないと効かない。`api := r.Group(...)` の `api` 側に `GET` を生やすこと。
- **`r.Run` の戻り error 無視**：ポート衝突に気づけない。握り潰さない。

## 関連
[middleware.md](./middleware.md) / [request_response.md](./request_response.md)
