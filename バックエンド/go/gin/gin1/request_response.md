# リクエスト / レスポンス（Request / Response）（Gin v1）

## ひとことで言うと
`gin.Context`(`c`) を通して「**入力を取り出す**」と「**応答を書く**」を行う一連のメソッド群。入力はパス/クエリ/フォーム/ヘッダ/ボディ、出力は JSON/String/XML/ステータス/リダイレクト。

## 役割・なぜ必要か
- HTTP の本質は「リクエストを読み、レスポンスを返す」こと。Gin はこの2方向の操作を `c` の短いメソッドにまとめ、標準ライブラリより簡潔に書けるようにしている。
- **入力の「どこから来た値か」でメソッドが分かれている**のが要点。場所（パス/クエリ/フォーム/JSON/ヘッダ）を取り違えると、値が空のまま気づかない不具合になる。
- 出力側は「**何を**返すか（JSON/テキスト/XML）」と「**どのステータスで**返すか」を明示する。ステータスを省くと既定で 200 になり、エラーが 200 で返る事故が起きる。

## 基本の書き方（コード）
```go
// ── 入力：取得元ごとにメソッドが違う ──────────────
func read(c *gin.Context) {
    id := c.Param("id")                       // パス /users/:id
    page := c.DefaultQuery("page", "1")       // クエリ。無ければ "1"
    keyword := c.Query("q")                   // クエリ。無ければ ""
    name := c.PostForm("name")                // application/x-www-form-urlencoded
    nick := c.DefaultPostForm("nick", "noname")
    token := c.GetHeader("Authorization")     // ヘッダ
    c.JSON(http.StatusOK, gin.H{
        "id": id, "page": page, "q": keyword, "name": name, "nick": nick, "token": token,
    })
}

// ── JSON ボディをバインドする ─────────────────────
type CreateUser struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

func create(c *gin.Context) {
    var in CreateUser
    if err := c.ShouldBindJSON(&in); err != nil {
        // 入力不正は 400 を明示
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // 201 を明示して作成結果を返す
    c.JSON(http.StatusCreated, gin.H{"name": in.Name, "email": in.Email})
}

// ── 出力いろいろ ──────────────────────────────────
func outputs(c *gin.Context) {
    switch c.Query("type") {
    case "text":
        c.String(http.StatusOK, "hello %s", c.Query("name")) // テキスト
    case "xml":
        c.XML(http.StatusOK, gin.H{"msg": "hi"})              // XML
    case "redirect":
        c.Redirect(http.StatusFound, "/login")               // 302 リダイレクト
    case "nobody":
        c.Status(http.StatusNoContent)                       // 本文なし(204)
    default:
        c.JSON(http.StatusOK, gin.H{"msg": "ok"})            // JSON
    }
}
```

## 実務での使い方・定番パターン
- **入力メソッドの早見表**：
  - パスパラメータ → `c.Param("id")`（`/users/:id`）
  - クエリ → `c.Query("q")` / 既定値つき `c.DefaultQuery("page","1")`
  - フォーム → `c.PostForm("name")` / `c.DefaultPostForm(...)`
  - JSONボディ → `c.ShouldBindJSON(&s)`（バインド＋検証は [binding_validation.md](./binding_validation.md)）
  - ヘッダ → `c.GetHeader("Authorization")`
- **値の有無を区別したい**：`v, ok := c.GetQuery("q")` / `c.GetPostForm("name")` は `(string, bool)` を返し、「空文字」と「未指定」を区別できる。
- **応答は必ずステータスを明示**：作成=`201`、入力不正=`400`、未認証=`401`、未検出=`404`、サーバ内=`500`。`net/http` の定数（`http.StatusCreated` 等）を使う。
- **応答は1回だけ**：`c.JSON` 等を書いた後に重ねて書かない。分岐後は `return` で抜ける。→ [context.md](./context.md)
- **JSONタグを必ず付ける**：`json:"email"` を付けないとフィールド名（先頭大文字の `Email`）でシリアライズされ、API契約とズレる。
- **`ShouldBind` 系を選ぶ**：`MustBindWith`/`Bind` 系は失敗時に自動で 400 を書いてしまい、自前のエラー整形ができない。`ShouldBindJSON` で受けて自分で応答する方が制御しやすい。

## ハマりどころ / アンチパターン
- **入力取得メソッドの取り違え**：JSONボディを `c.PostForm` で取ろうとして空になる、クエリを `c.Param` で取ろうとして空になる、が頻出。**送られてくる Content-Type と場所**に合わせる。
- **`Query` と `DefaultQuery` の混同**：未指定時に `Query` は `""`、`DefaultQuery` は既定値。空文字で分岐したいのに既定値で潰れる、の逆も起きる。「未指定」を判定したいなら `GetQuery` の `ok` を見る。
- **JSONタグ漏れ**：`json` タグが無いと出力キーが `Name`/`Email` のように大文字始まりになる。クライアントが小文字想定だと取りこぼす。
- **ステータスコードの省略**：`c.JSON(0, ...)` のつもりはなくても、別経路で `c.Status` を呼ばないと既定 200。エラー応答でも 200 が返り、クライアントが成功扱いする事故に。
- **ボディの二重読み**：`c.Request.Body` は一度読むと空になる。`ShouldBindJSON` した後に生ボディを再読する場合は `c.GetRawData()` 等で工夫が要る。
- **大文字小文字・URLエンコード**：ヘッダ名は大小無視だが、クエリ値は URL デコード済みで返る点に注意。

## 関連: [binding_validation.md](./binding_validation.md) / [context.md](./context.md)
