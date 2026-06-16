# DB（GORM / sqlx、内蔵なし）（Gin v1）

## ひとことで言うと
**Gin には DB 機能が無い**。Gin はあくまで HTTP ルーター + ミドルウェアで、データベース接続・ORM は**自分で選んで組み込む**。選択肢は **GORM（ORM・人気）** / **sqlx（薄い拡張）** / 標準 **`database/sql`** の3系統。

## 役割・なぜ必要か
- Web フレームワークと永続化層を**疎結合**にしておくため、Gin は DB を抱え込まない設計になっている。
- だから「どの DB ライブラリを使うか」「接続をどう管理するか」は**自分の責任**。ここを曖昧にすると接続リークや N+1 を踏む。
- ハンドラに SQL を直書きせず、**リポジトリ層**に隔離することで、テスト・差し替え・可読性が効く。

---

## 選択肢の比較
| ライブラリ | 立ち位置 | 向き |
|-----------|---------|------|
| **GORM** | フル機能 ORM。関連・マイグレーション・フックまで | 開発速度重視・関連が多い |
| **sqlx** | `database/sql` に構造体マッピングを足した薄い層 | SQL を自分で書きたい |
| `database/sql` | 標準。最も低レベル | 依存を増やしたくない |

## GORM の基本（接続とCRUD）
モデルを構造体で定義し、`*gorm.DB` を介して操作する。
```go
import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

type User struct {
    ID    uint   `gorm:"primaryKey"`
    Name  string
    Email string `gorm:"uniqueIndex"`
    Posts []Post // has many（関連）
}

func newDB(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }
    return db, nil
}

// CRUD
db.Create(&user)
db.First(&user, "email = ?", email) // ? でプレースホルダ（SQLインジェクション対策）
db.Model(&user).Update("name", "new")
db.Delete(&user, id)
```

## コネクションプール（必須設定）
`*gorm.DB` の下の `*sql.DB` を取り出して**プールの上限**を設定する。無設定だと負荷時に接続が枯渇・無制限に増える。
```go
sqlDB, _ := db.DB()
sqlDB.SetMaxOpenConns(25)                 // 同時に開く最大接続数
sqlDB.SetMaxIdleConns(5)                  // アイドルで保持する数
sqlDB.SetConnMaxLifetime(30 * time.Minute) // 接続の寿命（古い接続を捨てる）
```
`*gorm.DB`（や `*sql.DB`）は**アプリ起動時に1つ作って使い回す**。リクエストごとに開いてはいけない。

## リポジトリ層に分離（定番）
ハンドラから DB を直接触らず、インターフェース越しに使う。テスト時はモックに差し替えられる。
```go
type UserRepository interface {
    FindByID(ctx context.Context, id uint) (*User, error)
    Create(ctx context.Context, u *User) error
}

type gormUserRepo struct{ db *gorm.DB }

func (r *gormUserRepo) FindByID(ctx context.Context, id uint) (*User, error) {
    var u User
    // WithContext で context を伝播（タイムアウト・キャンセルが効く）
    if err := r.db.WithContext(ctx).First(&u, id).Error; err != nil {
        return nil, err
    }
    return &u, nil
}
```

## context.Context を渡す
リクエストの `context` を DB 呼び出しまで通すと、**クライアント切断やタイムアウトで途中キャンセル**できる。
```go
func handler(c *gin.Context) {
    // gin.Context ではなく Request の context を使う
    u, err := repo.FindByID(c.Request.Context(), 1)
    // ...
}
```
GORM なら `WithContext(ctx)`、`database/sql` なら `QueryContext(ctx, ...)` を使う。

## N+1 問題と Preload
一覧で関連を1件ずつ引くと SQL が `1 + N` 回走る。GORM では **`Preload`** でまとめて読む。
```go
// NG: ユーザーごとに Posts を都度クエリ → N+1
var users []User
db.Find(&users)
for _, u := range users {
    db.Where("user_id = ?", u.ID).Find(&u.Posts) // N 回走る
}

// OK: Preload で関連を一括ロード
db.Preload("Posts").Find(&users) // SELECT users / SELECT posts の2回で済む
```
さらに条件付きなら `Preload("Posts", "published = ?", true)`。JOIN で済むなら `Joins` も検討。

## トランザクション
複数の書き込みを**全部成功か全部取り消し**にまとめる。
```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err // return err でロールバック
    }
    if err := tx.Model(&stock).Update("count", stock.Count-1).Error; err != nil {
        return err
    }
    return nil // nil でコミット
})
```

## ハマりどころ / アンチパターン
- **「Gin が DB を持つ」と誤解**：内蔵は無い。**自分で選んで接続を作る**前提。
- **N+1（最頻）**：関連を都度引くと激遅。GORM は `Preload`、生 SQL なら JOIN や IN でまとめる。
- **コネクション管理ミス**：プール上限未設定で枯渇。リクエスト毎に `Open` するのは厳禁、**起動時に1つ**。
- **context 未伝播**：`context` を渡さないとタイムアウト・キャンセルが効かず、遅いクエリが詰まる。
- **文字列連結で SQL 組立**：`"WHERE id=" + id` はインジェクション。必ず `?` プレースホルダ。
- **エラー無視**：GORM は `.Error` にエラーを入れる。`First` 等の戻りで**毎回 `.Error` を確認**する。
- **ハンドラに SQL 直書き**：リポジトリ層へ隔離しないとテスト・差し替えが効かない。

## GORM 特有の現象と対策（N+1以外）
GORMは「ゼロ値の扱い」と「エラーが戻り値でなく `.Error`」が独特で、ここを知らないと静かにバグる。
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| **ゼロ値が更新されない** | `Updates(struct)` は構造体の**ゼロ値（`""`,`0`,`false`）を更新対象から除外**する。「在庫を0にする」「フラグをfalseに」が反映されない | 意図的にゼロへ更新するなら **`map[string]interface{}{"count": 0}`** か **`Select("count").Updates(...)`** |
| **論理削除なのに残る/出ない** | モデルに `gorm.DeletedAt` があると `Delete` は**ソフトデリート**になり、以降のクエリから自動で除外される | 物理削除は `Unscoped().Delete(...)`、削除済みも取るなら `Unscoped().Find(...)`。挙動を把握して使う |
| **`Find` が未発見でもエラーにならない** | `First`/`Last`/`Take` は未発見で `ErrRecordNotFound` を返すが、`Find`（スライス取得）は0件でもエラー無し | 「1件取得」は `First`＋`errors.Is(err, gorm.ErrRecordNotFound)`、`Find` は件数 `len()` で判定 |
| **エラーは例外でなく `.Error`** | Goに例外は無い。`db.First(&u)` の失敗は戻り値でなく `.Error` に入る | **毎回 `if err := db....Error; err != nil`** を確認（ハマりどころ「エラー無視」と同根） |
| `Save` が全カラムを上書き | `Save` は全フィールドをUPDATE（ゼロ値も含む） | 部分更新は `Updates`/`Update`、`Save` は「全項目を保存する」時だけ |
| 大量挿入/取得が遅い | ループで `Create` するとN回SQL | **`CreateInBatches(&rows, 100)`**、大量取得は **`FindInBatches`** で分割 |
| `Preload` と `Joins` の取り違え | `Preload`＝別クエリでまとめ取り（安全）。`Joins`＝SQL JOINで1クエリだが、**has-many では行が重複** | 多対1/1対1の絞り込みは `Joins`、コレクションは `Preload`。ネストは `Preload("Posts.Comments")` |

## 関連: [project_structure.md](./project_structure.md)
