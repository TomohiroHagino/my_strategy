# バインド / バリデーション（構造体タグ）（Gin v1）

## ひとことで言うと
**バインド** は、リクエスト（JSON / クエリ / フォーム）の値を **Goの構造体に流し込む**処理。**バリデーション** は、その際に構造体タグ `binding:"..."` で書いた条件（必須・形式）を自動チェックする仕組み。Gin は内部で **go-playground/validator** を使う。

## 役割・なぜ必要か
- 外部から来る生データ（信頼できない入力）を、**型付きの構造体**に変換して安全に扱うためにある。
- 「emailは必須でメール形式」「ageは0以上」といった検証を**手書きの if 連打ではなくタグ宣言**で済ませる。
- 入口（boundary）で弾くことで、サービス層以降は「検証済みの値」だけを相手にできる。

---

## 基本の書き方（コード）
リクエストの形を構造体で定義し、`json:` でキー名、`binding:` で検証条件を書く。
```go
type CreateUserReq struct {
    Name  string `json:"name"  binding:"required,min=2"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age"   binding:"gte=0,lte=130"`
}

func createUser(c *gin.Context) {
    var req CreateUserReq
    // JSONボディを req に流し込み、binding タグで検証する
    if err := c.ShouldBindJSON(&req); err != nil {
        // 失敗時は自分で 400 を返す（ShouldBind 系はステータスを触らない）
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"name": req.Name})
}
```

## バインド先ごとのメソッド
入力の場所によってメソッドを使い分ける。
```go
c.ShouldBindJSON(&req)   // Content-Type 無視で JSON として読む（API で最頻）
c.ShouldBindQuery(&req)  // ?page=2&limit=20 をクエリから読む（form タグを見る）
c.ShouldBind(&req)       // Content-Type を見て JSON/フォーム等を自動判別
c.ShouldBindUri(&req)    // /users/:id のパスパラメータを読む（uri タグ）
```
クエリ・フォームは `json:` ではなく **`form:` タグ**を見る点に注意。
```go
type ListReq struct {
    Page  int    `form:"page,default=1"  binding:"gte=1"`
    Limit int    `form:"limit,default=20" binding:"gte=1,lte=100"`
    Sort  string `form:"sort"             binding:"omitempty,oneof=asc desc"`
}
// c.ShouldBindQuery(&req) で ?page=2&sort=desc を受ける
```

## よく使う binding タグ
| タグ | 意味 |
|------|------|
| `required` | 必須（ゼロ値だとエラー） |
| `omitempty` | 空ならそれ以降の検証をスキップ（任意項目） |
| `email` / `url` / `uuid` | 形式チェック |
| `min` / `max` | 文字列長・スライス長・数値の範囲 |
| `gte` / `lte` / `gt` / `lt` | 数値の大小 |
| `oneof=asc desc` | 列挙のいずれか |
| `len=10` | 厳密な長さ |

## ShouldBind（自前処理） vs Bind（自動400）
**`ShouldBind*`** は検証失敗時に**エラーを返すだけ**でレスポンスを触らない。自分で `return` し、エラー整形して返す。**`Bind*`** は失敗すると**自動で 400 とエラーボディを書き込み**、内部で `c.Abort()` する。
```go
// Bind 系: 失敗で自動 400。だが return を書かないと処理が続行してしまう
func handler(c *gin.Context) {
    var req CreateUserReq
    if err := c.Bind(&req); err != nil {
        return // ← Bind は 400 を書くが、return しないと下の行が走る
    }
    // ...成功時の処理
}
```
実務では **`ShouldBind` 系 + 自前のエラー整形**が定番。レスポンス形式（エラー envelope）を統一できるため。

## カスタムバリデーション（任意）
標準タグで足りないときは validator を取り出して独自ルールを登録する。
```go
import "github.com/go-playground/validator/v10"

if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
    v.RegisterValidation("notblank", func(fl validator.FieldLevel) bool {
        return strings.TrimSpace(fl.Field().String()) != ""
    })
}
// 使う側: `binding:"notblank"`
```

## 必須 / 任意（ポインタの使い分け）
`required` は**ゼロ値を「未入力」とみなす**ため、「0 や false を明示的に受け取りたい」場合は**ポインタ**にして区別する。
```go
type UpdateReq struct {
    Active *bool `json:"active" binding:"omitempty"` // nil=未指定 / &false=明示的にfalse
    Score  *int  `json:"score"`                       // 0 と「未送信」を区別できる
}
// req.Active が nil なら更新しない、というロジックが書ける
```
`*bool` を使わず `bool` にすると、`false` と「キー自体が無い」が区別できない。

## ハマりどころ / アンチパターン
- **`binding` タグの書式**：条件は**カンマ区切り**（`required,email`）。スペース区切りは `oneof=a b` の値側だけ。
- **`Bind` で return 忘れ**：`Bind` は 400 を書くが処理は止まらない。**必ず `return`** する。
- **クエリに `json:` タグ**：`ShouldBindQuery` は `form:` を見る。`json:` だけだと値が入らない。
- **`required` とゼロ値**：`int` の 0、`bool` の false は「未入力」扱い。明示値が欲しいなら**ポインタ**へ。
- **エラーをそのまま返す**：`err.Error()` は validator の内部表現で**ユーザーに不親切**。本番はフィールド別に整形する。
- **ネスト構造体の検証漏れ**：入れ子の構造体も検証したいなら、親フィールドに `binding:"required"` だけでなく中身のタグも必要。

## 関連: [request_response.md](./request_response.md) / [error_handling.md](./error_handling.md)
