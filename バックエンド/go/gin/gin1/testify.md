# testify（Gin v1）

## ひとことで言うと
Go標準 `testing` の上に乗る**アサーション・モック・スイートのライブラリ**。`if w.Code != 200 { t.Errorf(...) }` を `assert.Equal(t, 200, w.Code)` の1行に置き換え、失敗時のメッセージも見やすく出す。`suite` で前後処理つきテストグループ、`mock` で依存のスタブが書ける。標準 `testing` / `httptest` を置き換えるのではなく**補強する**。

## 役割・なぜ必要か
- 標準 `testing` の `if 条件 { t.Errorf(...) }` は冗長で、期待値・実際値の差分も自分で整形することになる。`assert` 系を使うと**1行で書け、失敗時に「expected/actual」を自動表示**する。
- `assert`（失敗しても続行）と `require`（失敗したら即停止）を使い分けると、nil チェックの後に続く処理での nil 参照クラッシュを防げる。
- `mock.Mock` で repository / 外部クライアントをスタブ化でき、DBや外部APIに触れずにハンドラ・サービス層を検証できる。

## 基本の書き方（コード）
```bash
go get github.com/stretchr/testify
```
```go
// assert / require：Gin ハンドラを httptest で叩いて検証
import (
    "net/http"; "net/http/httptest"; "testing"
    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
)

func TestPing(t *testing.T) {
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/ping", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "pong"}) })

    w := httptest.NewRecorder()
    req, _ := http.NewRequest(http.MethodGet, "/ping", nil)
    r.ServeHTTP(w, req)

    assert.Equal(t, http.StatusOK, w.Code)          // expected, actual の順
    assert.Contains(t, w.Body.String(), "pong")     // 部分一致
    assert.JSONEq(t, `{"msg":"pong"}`, w.Body.String()) // JSON構造の一致
}
```
```go
// require：失敗したら即停止（後続の nil 参照クラッシュを防ぐ）
func TestUser(t *testing.T) {
    u, err := repo.FindUser(1)
    require.NoError(t, err)      // ここで失敗したら以降を実行しない
    require.NotNil(t, u)         // u が nil なら次行のクラッシュを回避
    assert.Equal(t, "Taro", u.Name)
}
```
```go
// suite：前後処理つきのテストグループ
import "github.com/stretchr/testify/suite"

type UserHandlerSuite struct {
    suite.Suite
    router *gin.Engine
}

func (s *UserHandlerSuite) SetupTest() {      // 各テスト前に毎回呼ばれる
    gin.SetMode(gin.TestMode)
    s.router = setupRouter()
}

func (s *UserHandlerSuite) TestList() {
    w := httptest.NewRecorder()
    req, _ := http.NewRequest(http.MethodGet, "/users", nil)
    s.router.ServeHTTP(w, req)
    s.Equal(http.StatusOK, w.Code)            // s.Equal は assert.Equal 相当
}

func TestUserHandlerSuite(t *testing.T) {     // go test から起動する入口
    suite.Run(t, new(UserHandlerSuite))
}
```
```go
// mock：repository をスタブ化してサービス層を単体検証
import "github.com/stretchr/testify/mock"

type MockRepo struct{ mock.Mock }

func (m *MockRepo) FindUser(id int) (*User, error) {
    args := m.Called(id)                       // 呼び出しを記録
    return args.Get(0).(*User), args.Error(1)
}

func TestService(t *testing.T) {
    repo := new(MockRepo)
    repo.On("FindUser", 1).Return(&User{Name: "Taro"}, nil) // 期待する呼び出しと戻り値
    svc := NewService(repo)

    u, err := svc.Get(1)
    require.NoError(t, err)
    assert.Equal(t, "Taro", u.Name)
    repo.AssertExpectations(t)                  // On で宣言した呼び出しが全部起きたか
}
```

## 実務での使い方・定番パターン
- **`assert` と `require` の使い分け**：基本は `assert`（複数の検証を続けて報告できる）。**nil/エラーチェックなど「これが失敗したら以降が無意味」な箇所は `require`**（即停止してクラッシュを防ぐ）。エラー受け取り直後は `require.NoError(t, err)` が定石。
- **引数順は `(t, expected, actual)`**：`assert.Equal(t, 200, w.Code)`。逆にすると失敗メッセージの expected/actual が入れ替わって混乱する。
- **JSONは `JSONEq` / `Contains`**：レスポンスボディは文字列一致だと空白・キー順で落ちる。構造一致なら `assert.JSONEq`、部分確認なら `assert.Contains`。
- **`suite` で共通セットアップ**：`SetupTest`（各テスト前）/ `SetupSuite`（スイート全体で1回）/ `TearDownTest` に DB接続やルータ生成をまとめる。`s.Equal` 等のメソッド形が使える。
- **`mock` は境界だけ**：repository・外部APIクライアントなど I/O 境界をスタブ化。`On(...).Return(...)` で期待を宣言し、`AssertExpectations(t)` で未呼び出しを検出。自前ロジックはモックしない。
- **テーブル駆動と併用**：標準のテーブル駆動（`t.Run` でサブテスト）の中で `assert` を使うと、ケース名つきで差分が読める。
- **httptest はそのまま**：testify は HTTP を叩く部分は提供しない。`httptest.NewRecorder()` ＋ `r.ServeHTTP(w, req)` の標準手順に assert を足す形（→ [testing.md](./testing.md)）。

## ハマりどころ
- **`assert` だけで nil 参照クラッシュ**：`assert.NotNil(t, u)` は失敗しても**続行**するため、次行で `u.Name` を触ってテストプロセスが panic で落ちる。nil チェックは `require.NotNil` にする。
- **expected/actual の順序ミス**：`assert.Equal(t, w.Code, 200)` のように逆順だと、失敗時メッセージの意味が反転する。常に expected を先に。
- **mock の `On` 引数不一致**：`On("FindUser", 1)` と宣言したのに実コードが `FindUser(2)` を呼ぶと「unexpected call」で落ちる。引数を問わないなら `mock.Anything` を使う。
- **`AssertExpectations` 忘れ**：これを呼ばないと「宣言したのに呼ばれなかった」モックを検出できず、テストが甘くなる。
- **`suite` のメソッド名規約**：テストメソッドは `Test` で始める必要がある。`SetupTest`（毎回）と `SetupSuite`（1回）の取り違えで状態リークが起きる。
- **`require` をゴルーチン内で使う**：`require` は `runtime.Goexit` で停止するため、別ゴルーチンで呼ぶと親に伝わらず想定外の挙動になる。並行コードでは結果をチャネルで集約してメインで検証する。

## 関連
[testing.md](./testing.md) / [project_structure.md](./project_structure.md) / [error_handling.md](./error_handling.md)
