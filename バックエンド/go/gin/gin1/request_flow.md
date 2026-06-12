# リクエストの流れ・各層は何を返すか（Gin v1）

## ひとことで言うと
1リクエストが **Router → Middleware → Handler → Service → Repository → DB** と降り、**Entity が逆向きに上がってきて、Handler が `c.JSON` で返す**。全層が `*gin.Context`(`c`) を共有し、入出力は `c` で行う。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（POSTなら request body = JSON）
   ▼
[Router]   URL+メソッド→Handlerを対応づけ／Middlewareを連鎖
   │
   ▼
[Middleware]  認証・ログなど横断処理（c.Next で次へ / c.Abort で停止）
   │
   ▼
[Handler]  c.ShouldBindJSON で入力DTOへバインド／Service を呼ぶ
   │
   ▼
[Service]  業務ロジック／Repository を呼ぶ
   │
   ▼
[Repository]  DBアクセス（GORM / sqlx）
   │
   ▼
  DB ──→ Entity(構造体) を返す ─┐
   ▲                            │
[Repository] が Entity を Service に返す
   ▲
[Service]    が Entity / DTO を Handler に返す
   ▲
[Handler]    が c.JSON(...) でレスポンスを書き込む
   │ レスポンス（response body = JSON）
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **Router** | メソッド + URL | 対応するHandler/Middleware連鎖へ振り分け |
| **Middleware** | `*gin.Context` | `c.Next()`で続行 / `c.Abort()`で停止 |
| **Handler** | `*gin.Context`（bindで入力DTO化） | `c.JSON` 等でレスポンスを書く（戻り値なし） |
| **Service** | DTO / 値 | **Entity or DTO**（Handlerへ） |
| **Repository** | id / 条件 / 保存データ | **Entity**（構造体、DBから） |

- Goは戻り値で `(value, error)` を返すのが基本。各層は値とerrorを返し、Handlerがerrorを見て `c.JSON` で出力。
- Handler自身は値を返さず、`c.JSON`/`c.AbortWithStatusJSON` でレスポンスへ直接書き込む。

## コードで通して見る
```go
// 1) Handler：bindで入力DTO化 → Service呼び出し → c.JSONで返す
func CreatePost(c *gin.Context) {
    var req CreatePostRequest
    if err := c.ShouldBindJSON(&req); err != nil {   // 入力検証（構造体タグ）
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    post, err := postService.Create(req)             // ServiceがEntityを返す
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, post)                      // レスポンスを書き込む
}

// 2) Service：業務処理 → Repository呼び出し → Entityを返す
func (s *PostService) Create(req CreatePostRequest) (*Post, error) {
    post := &Post{Title: req.Title, Body: req.Body}  // DTO→Entity
    return s.repo.Save(post)                          // RepositoryがEntityを返す
}

// 3) Repository：DBアクセス。Entityを返す
func (r *PostRepository) Save(p *Post) (*Post, error) {
    if err := r.db.Create(p).Error; err != nil { return nil, err }
    return p, nil
}
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：Repo→Service は Entity、Service→Handler は Entity/DTO、Handler→クライアントは `c.JSON`。
- **handler/service/repository でディレクトリ分割**：依存はコンストラクタ注入。→ [project_structure.md](./project_structure.md)
- **入力検証は構造体タグ + ShouldBind**：`binding:"required"` などで境界を固める。→ [binding_validation.md](./binding_validation.md)
- **errorは握りつぶさず上へ返す**：各層は `(value, error)` を返し、Handlerで一括して `c.JSON`。→ [error_handling.md](./error_handling.md)

## ハマりどころ / アンチパターン
- **Handlerに業務やDBアクセスを直書き**：層が崩れる。Service/Repositoryへ。
- **`c.JSON` 後に処理を続ける**：レスポンスは1回。書いたら `return` する。→ [context.md](./context.md)
- **`c.Abort` 忘れ**：Middlewareで拒否したつもりが後続Handlerが動く。→ [middleware.md](./middleware.md)
- **errorを無視（`_`）**：DBエラーが闇に消える。必ず返して扱う。→ [pitfalls.md](./pitfalls.md)

## 関連
[routing.md](./routing.md) / [middleware.md](./middleware.md) / [context.md](./context.md) / [request_response.md](./request_response.md) / [binding_validation.md](./binding_validation.md) / [project_structure.md](./project_structure.md)
