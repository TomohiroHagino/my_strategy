# プロジェクト構成（Gin v1）

## ひとことで言うと
Gin アプリのコードをどう分割・配置するか。**Gin / Go には Rails のような公式の規約は無い**ので、チームで決めた「レイヤリング」を自分で持ち込む。

## 役割・なぜ必要か
- Gin は「ルータとミドルウェア」だけを提供する。ディレクトリ構成・DB アクセス・依存注入は**全部こちら任せ**。放っておくと `main.go` に全部書かれて肥大化する。
- 役割ごとに層を分けると、テスト・差し替え・読みやすさが効いてくる。定番は **handler → service → repository** の3層。
  - **handler**：HTTP の入口。リクエストを受けて検証し、service を呼び、レスポンスを返す（HTTP の都合だけを扱う）。
  - **service**：業務ロジック。複数 repository をまたぐ手続きや判断はここ。HTTP も SQL も知らない。
  - **repository**：データアクセス。DB / 外部 API の出し入れだけ。
- 1方向の依存（handler→service→repository）にすることで、上位を下位のモックで差し替えてテストできる。

## 基本の書き方（コード）
```bash
myapp/
├── cmd/
│   └── server/
│       └── main.go        # エントリポイント。配線（DI）だけ書く
├── internal/              # このモジュール外から import 不可（隠す）
│   ├── handler/
│   │   └── user.go        # UserHandler（*gin.Context を触る層）
│   ├── service/
│   │   └── user.go        # UserService（業務ロジック）
│   ├── repository/
│   │   └── user.go        # UserRepository（DB アクセス）
│   └── model/
│       └── user.go        # ドメイン構造体
├── pkg/                   # 外部公開してよい汎用ユーティリティ
├── go.mod
└── go.sum
```

```go
// cmd/server/main.go — 依存は「手動で」注入する（DIフレームワーク不要）
func main() {
	db := mustOpenDB()

	userRepo := repository.NewUserRepo(db)
	userSvc := service.NewUserService(userRepo)
	userH := handler.NewUserHandler(userSvc)

	r := gin.Default()
	api := r.Group("/api")
	{
		api.GET("/users/:id", userH.Get)
		api.POST("/users", userH.Create)
	}
	r.Run(":8080")
}
```

```go
// internal/handler/user.go — handler は service にだけ依存
type UserHandler struct{ svc *service.UserService }

func NewUserHandler(svc *service.UserService) *UserHandler {
	return &UserHandler{svc: svc}
}

func (h *UserHandler) Get(c *gin.Context) {
	u, err := h.svc.FindByID(c.Param("id"))
	if err != nil {
		c.JSON(404, gin.H{"error": "not found"})
		return
	}
	c.JSON(200, u)
}
```

## 実務での使い方・定番パターン
- **`cmd/` にエントリ、`internal/` に本体**：`cmd/server/main.go` は「配線だけ」に保つ。複数バイナリ（`cmd/server`, `cmd/worker`）も置ける。
- **`internal/` で外部 import を遮断**：Go は `internal/` を特別扱いし、そのモジュールの外からは import できない。公開 API として固める前のコードを隠すのに使う。
- **`pkg/` は外に出してよい汎用品だけ**：迷ったら `internal/` に置く。`pkg/` は安易に作らない方が良いという意見も多い。
- **依存は interface で受ける**：service が `repository` の interface を受け取れば、テストでモックに差し替えられる。
- **小さく始める**：最初から3層フルに割らず、規模が出てから service / repository を切り出すのも YAGNI 的に妥当。

## ハマりどころ / アンチパターン
- **巨大 `main.go`**：ルート定義・DB 接続・ハンドラ実装を全部 `main()` に詰める。配線とロジックを分け、`internal/` 以下へ出す。
- **`internal/` の意味を誤解**：`internal/` は「private なフォルダ名」ではなく Go のルール（外部モジュールから import 不可）。同一モジュール内からは普通に使える。
- **循環 import**：handler が service を、service が handler を import…のように相互参照すると Go はコンパイルエラーにする。依存は**一方向**に保ち、共有する型は `model/` など下位に置いて解消する。
- **「型でフォルダを切る」過剰分割**：`controllers/` `services/` を機能に関係なく増やすより、ドメイン（`user/` `order/`）単位の方が見通しが良いことが多い。
- **DI フレームワークの早すぎる導入**：Go では手動の配線（コンストラクタ注入）で十分なことが大半。`wire` 等は規模が出てから検討する。

## 関連
[routing.md](./routing.md) / [database.md](./database.md)
