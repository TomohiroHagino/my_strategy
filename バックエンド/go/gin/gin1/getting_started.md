# 始め方（Gin v1）

## ひとことで言うと
Gin プロジェクトを動かすための最小手順。**`go mod` でモジュールを作り → Gin を `go get` し → `gin.Default()` でルータを起動する**、ここまでが土台。

## 役割・なぜ必要か
- Go は「モジュール（`go.mod`）」単位で依存を管理する。最初に `go mod init` をしておかないと外部ライブラリを取り込めない。
- Gin は標準ライブラリ `net/http` の薄いラッパーで、ルーティング・ミドルウェア・JSON 返却などを書きやすくする。素の `net/http` でも書けるが、ルーティングやミドルウェアを自前で組むと冗長になる。
- `gin.Default()` は **Logger（アクセスログ）＋ Recovery（panic を 500 に変換）** が最初から入った状態でルータを返す。まず動かして体感するならこれが入口。

## 基本の書き方（コード）
```bash
# 1) プロジェクト用ディレクトリを作り、モジュールを初期化
mkdir myapp && cd myapp
go mod init github.com/yourname/myapp   # モジュールパスは任意（リポジトリURL推奨）

# 2) Gin を取得（go.mod / go.sum に記録される）
go get github.com/gin-gonic/gin

# 3) 実行
go run .
```

```go
// main.go
package main

import "github.com/gin-gonic/gin"

func main() {
	// Logger + Recovery 入りのルータ
	r := gin.Default()

	// "/" への GET。c は *gin.Context（入出力の窓口）
	r.GET("/", func(c *gin.Context) {
		// gin.H は map[string]any のショートハンド
		c.JSON(200, gin.H{"msg": "ok"})
	})

	// :8080 で待ち受け（戻り値の error は握り潰さない）
	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

```bash
# 動作確認
curl http://localhost:8080/
# => {"msg":"ok"}
```

## 実務での使い方・定番パターン
- **ホットリロードは `air`** を使うと開発が速い。ファイル保存で自動再ビルド・再起動される。
  ```bash
  go install github.com/air-verse/air@latest   # $GOPATH/bin に入る
  air init                                       # .air.toml を生成
  air                                            # 以後 air で起動
  ```
  `air` が `command not found` のときは `$GOPATH/bin`（通常 `~/go/bin`）を `PATH` に追加する。
- **`gin.Default()` と `gin.New()` を使い分ける**：本番では `gin.New()` から始めて、必要なミドルウェアだけ明示的に積む構成が好まれる（後述）。
- **ポートは環境変数から**読むと本番（PaaS など）で扱いやすい。
  ```go
  port := os.Getenv("PORT")
  if port == "" {
  	port = "8080"
  }
  r.Run(":" + port)
  ```
- **本番モードに切り替える**：`gin.SetMode(gin.ReleaseMode)`（または環境変数 `GIN_MODE=release`）でデバッグログを抑制する。設定しないと起動時に警告が出る。

## ハマりどころ / アンチパターン
- **`go mod init` を忘れる / モジュールパスが雑**：`go get` で「go.mod が無い」とエラーになる。モジュールパスはリポジトリURL（`github.com/...`）に合わせておくと後で import がぶれない。
- **`gin.Default()` と `gin.New()` の違いを知らない**：
  - `gin.Default()` = `gin.New()` + Logger + Recovery。
  - `gin.New()` = **素のルータ**（ミドルウェアゼロ）。これだけだと panic で接続が落ち、ログも出ない。`gin.New()` を使うなら `r.Use(gin.Logger(), gin.Recovery())` を自分で積む。
- **ポート衝突（`bind: address already in use`）**：前のプロセスが残っている。`lsof -i :8080` で確認して `kill`、または別ポートで起動する。
- **`r.Run()` の戻り error を捨てる**：ポート確保失敗などに気づけない。最低でも `panic` か `log.Fatal` で握り潰さない。
- **`go run main.go` だけ叩いて他ファイルを見落とす**：複数ファイル構成なら `go run .`（ディレクトリ指定）が安全。

## フォルダ構成（始動直後）
```
myapp/                                    # ※Ginに公式雛形は無い。golang-standards 準拠の一例
├── cmd/
│   └── server/
│       └── main.go                       # エントリ（router構築・起動）
├── internal/                             # 外部importを禁止する非公開パッケージ
│   ├── handler/                          # = controller。HTTP受付（gin.Context）
│   │   └── user_handler.go
│   ├── service/                          # 業務ロジック
│   │   └── user_service.go
│   ├── repository/                       # DBアクセス
│   │   └── user_repository.go
│   ├── model/                            # 構造体（ドメイン/DB）
│   │   └── user.go
│   ├── middleware/                       # 認証・ログ等
│   │   └── auth.go
│   └── router/
│       └── router.go                     # ルーティング定義
├── config/
│   └── config.go                         # 設定読み込み（env/yaml）
├── pkg/                                  # 外部公開してよい共有コード（任意）
├── migrations/                           # SQLマイグレーション（任意）
├── go.mod  go.sum                        # 依存管理（go mod init で生成）
├── .env                                  # 環境変数
├── Makefile                              # build/run/test ショートカット
└── Dockerfile                            # コンテナ化（任意）
```
- **標準のディレクトリ構成は無い。** `main.go` 1枚から始めてもよいが、上は広く使われる **cmd/ + internal/ の層分け**。
- `internal/` は外部リポジトリから import 不可＝公開したくない実装を隔離する Go の仕組み。
- 層（handler/service/repository）は Spring と同発想。基本すべて自分で作る。

## 関連
[routing.md](./routing.md) / [middleware.md](./middleware.md)
