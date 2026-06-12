# テスト（Testing）（Gin v1）

## ひとことで言うと
ハンドラやルーティングの振る舞いを**自動で検証するコード**。Goは標準の `testing` パッケージ＋ **`net/http/httptest`** で、サーバを実際に立てずに HTTP リクエストを再現して検証する。

## 役割・なぜ必要か
- 変更のたびに手で全エンドポイントを叩くのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- httptest は実ポートを開かず**メモリ上でリクエスト→ハンドラ→レスポンス**を完結させるので、速くて安定。
- ルーティング・バインド・ステータス・JSON本文まで一気通貫で検証でき、「動く仕様書」になる。

## 基本の書き方（コード）
### 最小：httptest でハンドラを叩く
```go
import (
    "net/http"; "net/http/httptest"; "testing"
    "github.com/gin-gonic/gin"
)

func setupRouter() *gin.Engine {
    gin.SetMode(gin.TestMode) // テスト時はTestMode（デバッグログを抑制）
    r := gin.New()
    r.GET("/ping", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "pong"}) })
    return r
}

func TestPing(t *testing.T) {
    r := setupRouter()
    w := httptest.NewRecorder()                       // レスポンス記録
    req, _ := http.NewRequest(http.MethodGet, "/ping", nil)
    r.ServeHTTP(w, req)                               // 実サーバ不要で実行

    if w.Code != http.StatusOK {
        t.Errorf("status = %d, want 200", w.Code)
    }
    if !strings.Contains(w.Body.String(), "pong") {
        t.Errorf("body = %s", w.Body.String())
    }
}
```

### POST + JSON ボディ
```go
func TestCreate(t *testing.T) {
    r := setupRouter()
    body := bytes.NewBufferString(`{"name":"Taro"}`)
    req, _ := http.NewRequest(http.MethodPost, "/users", body)
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    if w.Code != http.StatusCreated {
        t.Errorf("status = %d, want 201", w.Code)
    }
}
```

### テーブル駆動テスト（定番）
```go
func TestValidation(t *testing.T) {
    r := setupRouter()
    tests := []struct {
        name     string
        body     string
        wantCode int
    }{
        {"正常", `{"name":"Taro"}`, http.StatusCreated},
        {"name欠落", `{}`, http.StatusBadRequest},
        {"不正JSON", `{`, http.StatusBadRequest},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) { // サブテストで個別に可視化
            w := httptest.NewRecorder()
            req, _ := http.NewRequest(http.MethodPost, "/users",
                bytes.NewBufferString(tt.body))
            req.Header.Set("Content-Type", "application/json")
            r.ServeHTTP(w, req)
            if w.Code != tt.wantCode {
                t.Errorf("%s: code = %d, want %d", tt.name, w.Code, tt.wantCode)
            }
        })
    }
}
```

## 実務での使い方・定番パターン
- **ルータ生成を関数に切り出す**（`setupRouter()`）。各テストで使い回し、ミドルウェアも本番同等にして結合を検証する。
- **テーブル駆動**で正常系・異常系・境界値をまとめて回す。`t.Run` でケース名を付け、失敗箇所を特定しやすくする。
- **テストDBを分離**：本番DBを汚さないよう専用DBを使い、各テストでトランザクションをロールバック or テーブルをクリーンアップ。SQLite in-memory や testcontainers も定番。
- 認証必須ルートは、**ミドルウェアをモックする**か、テスト用トークンを `Authorization` ヘッダに付けて叩く。
- カバレッジは `go test -cover ./...` で計測。重要フロー（作成/更新/認可）を優先的に埋める（→ [project_structure.md](./project_structure.md)）。

## ハマりどころ / アンチパターン
- **`gin.TestMode` 未設定**：`gin.Default()` のままだと大量のデバッグログが出てテスト出力が読みづらい。`gin.SetMode(gin.TestMode)` を入れる。
- **実サーバを立ててテスト**：`r.Run()` してポートを開くとポート競合・後始末漏れで不安定。**`r.ServeHTTP(w, req)`** でメモリ内完結させる。
- **テストDBを分離しない**：本番/開発DBを共有するとデータが壊れ、テスト間で状態がリークして順序依存で落ちる。専用DB＋毎回クリーンアップ。
- **`Content-Type` 付け忘れ**：JSONボディでも `application/json` を付けないと `ShouldBindJSON` がパースせず、想定外の結果に。
- **レスポンスボディ未検証**：ステータスだけ見て本文を確認しないと、誤った内容を返していても緑になる。本文/JSONも検証する。
- **グローバル状態の共有**：パッケージ変数のキャッシュ等が残り flaky に。各テストで初期化する。

## 関連
[testify.md](./testify.md) / [project_structure.md](./project_structure.md)
