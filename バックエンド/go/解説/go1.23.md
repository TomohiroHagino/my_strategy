# Go 1.23（言語解説）

## ひとことで言うと
Google 製の静的型付けコンパイル言語で、構造体とインターフェース（暗黙実装）・エラー値・goroutine/channel による並行処理を軸にした「誰が書いても似た形になる」言語。1.23 の目玉は `for range` で関数（イテレータ）を直接回せる range-over-func の正式化。

## このバージョンの位置づけ（リリース / サポート・LTS / どこで使うか）
- リリース: 2024年8月。Go に LTS という概念はなく、約半年ごとにマイナー版が出て、直近 2 マイナー版にセキュリティ修正が提供される。常に新しめを追うのが基本。
- 後方互換は強く保証される（Go 1 compatibility promise）。1.23 で書いたコードは将来版でも基本動く。
- 使う場所: API / マイクロサービス（高並行・低レイテンシ）、CLI・インフラツール（Docker・Kubernetes・Terraform 自体が Go 製）。Web フレームワークは [../gin/](../gin/) を参照。

```bash
go version          # go version go1.23.x ...
go mod init example.com/app   # モジュール初期化（go.mod 生成）
go run .            # ビルドして即実行
go build -o app .   # 単一の静的バイナリを生成
```

## 言語の基本（文法の要点）
パッケージ単位で構成し、`main` パッケージの `main` 関数がエントリポイント。型は変数名の後ろに書く。

```go
package main

import "fmt"

func main() {
    var count int = 3       // 明示
    name := "Go"            // := で型推論＋宣言（関数内のみ）
    for i := 0; i < count; i++ {
        fmt.Printf("%s%d\n", name, i)
    }
}
```

- セミコロンは不要（コンパイラが自動挿入）。`{` は同じ行に置く決まり。
- 未使用の変数・import はコンパイルエラー。`gofmt` で整形が一意に決まる。
- 関数は複数戻り値を返せる。慣習として最後の戻り値にエラーを置く。

```go
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}
```

## この言語の核心概念（他言語と違う・必ず押さえる）

### ポインタ（`&` で取得・`*` でデリファレンス）
- **何か**: 値が置かれているメモリ上の住所を持つ型。`&x` で住所を取り、`*p` でその住所の中身を読み書きする。住所を入れていない状態は `nil`。
- **具体コード**:

```go
x := 10
p := &x        // p は *int（x の住所）
fmt.Println(*p) // 10（デリファレンス＝中身を読む）
*p = 20         // 住所経由で書き込む
fmt.Println(x)  // 20（x 本体が変わる）

var q *int      // 何も指していないポインタ
fmt.Println(q == nil) // true
// fmt.Println(*q)    // nil をデリファレンスすると panic

func increment(n *int) { // 関数引数で「呼び出し元の変数を変えたい」時に使う
    *n++
}
y := 5
increment(&y)   // y の住所を渡す
fmt.Println(y)  // 6
```

- **他言語と違う点/つまずき**: Java/JS/Python のように「参照かどうか」が言語に隠蔽されておらず、`&`/`*` を自分で書く。`nil` ポインタのデリファレンスは即 panic。Go には C のようなポインタ演算（`p+1`）は無く、安全な範囲に限定される。

### 値型 vs ポインタのセマンティクス
- **何か**: 構造体・配列・基本型は **代入・関数引数渡しでコピーされる**。コピーを変更しても元には反映されない。元を変えたいならポインタを渡す。メソッドのレシーバも同じ理屈で「値レシーバ（コピー）」と「ポインタレシーバ（本体）」を使い分ける。
- **具体コード**:

```go
type Counter struct{ n int }

func (c Counter) IncValue()  { c.n++ } // 値レシーバ＝コピーを変更（呼び出し元に反映されない）
func (c *Counter) IncPtr()   { c.n++ } // ポインタレシーバ＝本体を変更

func main() {
    c := Counter{}
    c.IncValue()
    fmt.Println(c.n) // 0（コピーを変えただけ）
    c.IncPtr()       // Go が自動で (&c).IncPtr() に変換
    fmt.Println(c.n) // 1（本体が変わった）

    d := c           // 構造体は代入でまるごとコピー
    d.n = 99
    fmt.Println(c.n) // 1（d を変えても c は無関係）
}
```

- **他言語と違う点/つまずき**: Java/JS では「オブジェクトを渡す＝参照のコピー」で中身は常に共有されるが、Go の構造体は **中身ごとコピー** される（JS のオブジェクトとは逆の感覚）。値レシーバのメソッドで状態を変えても無意味、という事故が定番。**変更するメソッドはポインタレシーバ** が原則。一つの型でレシーバを混在させない。

### スライスの内部（配列へのビュー＝ptr/len/cap）
- **何か**: スライスは「背後の配列を指すポインタ・長さ `len`・容量 `cap`」の3点組。**配列へのビュー** なので、同じ配列を指す複数のスライスは互いに影響する。`append` は `cap` を超えると新しい配列を確保してコピーする（＝再確保後は元と切り離される）。
- **具体コード**:

```go
base := []int{1, 2, 3, 4, 5}
s := base[1:3]           // [2 3] len=2 cap=4（base を共有するビュー）
s[0] = 99
fmt.Println(base)        // [1 99 3 4 5] ← base 側も書き換わる

a := make([]int, 0, 2)   // len=0 cap=2
a = append(a, 1)         // cap 内なので同じ配列
b := a
b = append(b, 2)         // まだ cap 内 → a の背後配列も触る可能性
b = append(b, 3)         // cap 超過 → 再確保。ここから b は a と別配列
```

- **他言語と違う点/つまずき**: スライスをコピー（代入・引数渡し）しても **背後の配列は共有** される。「別物のつもりで書き換えたら元も変わった」が頻出バグ。逆に `append` で再確保が起きると突然共有が切れるため、共有を当てにしたコードは壊れる。安全に独立させたいなら `copy()` で別配列に複製する。

### map とゼロ値
- **何か**: Go の変数は宣言しただけで型ごとの **ゼロ値** が入る（数値0・文字列`""`・bool`false`・ポインタ/スライス/map `nil`）。`var m map[string]int` の段階では `nil` map で、**読み取りはできるが書き込むと panic**。`make` か リテラルで初期化が必要。
- **具体コード**:

```go
var m map[string]int
fmt.Println(m["x"]) // 0（無いキーはゼロ値が返る・panic しない）
// m["x"] = 1       // panic: assignment to entry in nil map

m = make(map[string]int) // 初期化すれば書ける
m["x"] = 1

v, ok := m["y"]     // ok でキーの有無を判定（v はゼロ値0）
fmt.Println(v, ok)  // 0 false
```

- **他言語と違う点/つまずき**: 「無いキーを引くと例外/`undefined`」ではなく **ゼロ値が返る** ため、値が0なのかキーが無いのかは `v, ok :=` 形式で区別する。`nil` map への書き込み panic は初心者の典型エラー。null/undefined が無い代わりに、すべての型に決まったゼロ値がある点が他言語と大きく違う。

### 暗黙インターフェース／コンポジション（継承なし）
- **何か**: インターフェースは「必要なメソッドを満たしていれば自動的に実装したとみなす」（`implements` 宣言が不要＝構造的）。継承は無く、型の埋め込み（コンポジション）で機能を合成する。
- **具体コード**:

```go
type Reader interface{ Read() string }

type File struct{ name string }
func (f File) Read() string { return "data of " + f.name } // File は自動的に Reader

// 埋め込み（継承ではなく合成）
type Logger struct{ prefix string }
func (l Logger) Log(msg string) { fmt.Println(l.prefix, msg) }

type Service struct {
    Logger   // 埋め込み: Service は Log を「持つ」ように振る舞う
    name string
}

func main() {
    var r Reader = File{name: "a.txt"} // 明示実装ゼロで代入できる
    fmt.Println(r.Read())
    s := Service{Logger: Logger{prefix: "[svc]"}}
    s.Log("started") // 埋め込んだ Logger のメソッドをそのまま呼べる
}
```

- **他言語と違う点/つまずき**: Java の `implements`/`extends` のような明示宣言が無いため、「どのインターフェースを満たすか」がコード上に書かれない（型を見ただけでは分かりにくい）。`is-a` の継承ではなく `has-a` の埋め込みで考える。埋め込みはオーバーライドではなく委譲に近い点に注意。

## 型・データモデル

### 基本型（全部に具体例）
```go
var i  int     = 42            // 整数（環境依存で32/64bit。普段はこれ）
var i64 int64  = 9000000000    // 明示64bit。int32/int16/int8 もある
var u  uint    = 7             // 符号なし整数（0以上のみ）
var f  float64 = 3.14          // 浮動小数点（普段はこれ。float32 もある）
var b  bool    = true          // 真偽値
var s  string  = "こんにちは"   // 文字列（UTF-8・イミュータブル）

// byte は uint8 の別名（=1バイト）。ASCII やバイナリを扱うとき
var by byte = 'A'              // 65。fmt.Println(by) は 65 を出す
// rune は int32 の別名（=Unicodeコードポイント1文字）。日本語など多バイト文字用
var r  rune = '本'             // 26412

// 文字列を「バイト単位」か「文字単位」かで結果が変わる（定番のつまずき）
s2 := "abcあ"
fmt.Println(len(s2))            // 6 … バイト数（"あ"は3バイト）
fmt.Println(len([]rune(s2)))    // 4 … 文字数。日本語を数えるには []rune に変換
for i, c := range s2 {          // range は rune 単位で回る
    fmt.Printf("%d=%c ", i, c)  // 0=a 1=b 2=c 3=あ（インデックスはバイト位置）
}
```

### 複合型（全部に具体例）
```go
// ① 配列（固定長）：長さが型の一部。長さ違いは別の型
var arr [3]int = [3]int{10, 20, 30}
arr[0] = 99                    // arr は [99 20 30]
// arr2 := [4]int{} は arr とは別の型（代入不可）。だから実務ではほぼスライスを使う

// ② スライス（可変長）：実務の主役。背後の配列へのビュー
sl := []int{1, 2, 3}           // [] に長さを書かない＝スライス
sl = append(sl, 4)             // [1 2 3 4]
sl2 := sl[1:3]                 // [2 3]（部分スライス。同じ配列を共有する）
sl3 := make([]int, 0, 10)      // 長さ0・容量10で先に確保（appendの再確保を減らす）

// ③ マップ（キー→値）
m := map[string]int{"a": 1, "b": 2}
m["c"] = 3                     // 追加
v, ok := m["a"]                // v=1, ok=true（存在確認は必ずこの2値で）
_, ok2 := m["x"]               // ok2=false（無いキーはゼロ値が返るので ok で判定）
delete(m, "a")                 // 削除

// ④ 構造体（フィールドの集まり）
type User struct {
    ID   int
    Name string
}
u := User{ID: 1, Name: "alice"}      // フィールド名指定
u2 := User{2, "bob"}                 // 順番指定（非推奨：順序変更に弱い）
fmt.Println(u.Name)                  // alice

// ⑤ ポインタ（値の住所）
p := &u                        // &でアドレスを取る → *User型
p.Name = "carol"               // ポインタ経由で本体を変更（自動でデリファレンス）
fmt.Println(u.Name)            // carol（uそのものが変わった）
fmt.Println(*p)                // *でデリファレンス → {1 carol}
```

### ゼロ値（宣言しただけで入る初期値・全型）
Goは「未初期化」が無い。`var` で宣言した瞬間、型ごとの**ゼロ値**が必ず入る。
```go
var i int        // 0
var f float64    // 0
var b bool       // false
var s string     // ""（空文字。nil ではない）
var p *int       // nil（何も指さないポインタ）
var sl []int     // nil（長さ0。ただし append はできる）
var m map[string]int // nil（読めるが【書くと panic】→ make が必要）
var st User      // {0 ""}（各フィールドがゼロ値の構造体）

fmt.Println(i, f, b, s == "", p == nil, sl == nil, m == nil)
// 0 0 false true true true true

// よくある事故：nil マップに書き込むと実行時 panic
var bad map[string]int
// bad["x"] = 1            // panic: assignment to entry in nil map
good := make(map[string]int) // make すれば書ける
good["x"] = 1
```
- **ポイント**：`nil` スライスは `append` できるが、`nil` マップは**書き込み不可**（読みはOK）。この非対称が定番のハマり所。

（暗黙インターフェース・コンポジションは「[この言語の核心概念](#この言語の核心概念他言語と違う必ず押さえる)」を参照）

### ジェネリクス（1.18+ / 1.23 で成熟）
型パラメータ `[T any]` と制約で書く。標準ライブラリにも `slices` / `maps` パッケージが揃った。

```go
func Map[T, U any](in []T, f func(T) U) []U {
    out := make([]U, len(in))
    for i, v := range in {
        out[i] = f(v)
    }
    return out
}

doubled := Map([]int{1, 2, 3}, func(n int) int { return n * 2 }) // [2 4 6]
```

## この言語らしさ / 特徴的な機能
### 明示的なエラー処理
例外機構を使わず、エラーを値として返して `if err != nil` で都度処理する。

```go
f, err := os.Open("config.json")
if err != nil {
    return fmt.Errorf("open config: %w", err) // %w でラップして文脈を追加
}
defer f.Close()                                // 関数終了時に必ず実行
```

```go
// ラップしたエラーの判別
if errors.Is(err, os.ErrNotExist) { /* ファイルが無い */ }
var pathErr *os.PathError
if errors.As(err, &pathErr) { /* 具体型を取り出す */ }
```

### defer / panic / recover
`defer` は後始末の定番。`panic` は想定外の致命的状況に限り、`recover` で回収できる（通常のエラーには使わない）。

## 並行・非同期
goroutine は数KB から始まる超軽量な並行処理単位。`go` を付けるだけで起動し、channel で安全に値を受け渡す。

```go
ch := make(chan int)
go func() {            // goroutine 起動
    ch <- 42           // 送信
}()
v := <-ch              // 受信（届くまでブロック）
```

```go
// 複数 goroutine の完了待ち
var wg sync.WaitGroup
for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("worker", id)
    }(i)
}
wg.Wait()
```

```go
// select で複数 channel を待つ / タイムアウト
select {
case v := <-ch:
    fmt.Println("got", v)
case <-time.After(time.Second):
    fmt.Println("timeout")
}
```

- 「メモリを共有して通信するな、通信してメモリを共有せよ」が設計思想。共有状態には `sync.Mutex` も使う。
- `context.Context` でキャンセル・期限をまたいで伝播させるのが定番。

## 標準ライブラリ / ツールチェーン
- モジュール: `go mod`（`go.mod` / `go.sum`）。`go get` で依存追加。
- 整形/解析: `go fmt`（=`gofmt`）、`go vet`、`golangci-lint`。
- テスト: 標準 `testing`（テーブル駆動が定番）。

```go
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    int
        want    int
        wantErr bool
    }{
        {"ok", 6, 2, 3, false},
        {"zero", 1, 0, 0, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := divide(tt.a, tt.b)
            if (err != nil) != tt.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tt.wantErr)
            }
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}
```

```bash
go test ./... -cover    # 全パッケージをテスト＋カバレッジ
```

- 厚い標準ライブラリ: `net/http`（Web サーバ/クライアント）、`encoding/json`、`io`、`os`、`time`。

## このバージョンの新機能・トピック
### range-over-func（イテレータ正式化, 1.23の目玉）
`for range` に「イテレータ関数」を直接渡せるようになった。`func(yield func(T) bool)` 型を返せば独自の反復を `for range` で回せる。

```go
// 1..n を生成するイテレータ
func Count(n int) func(yield func(int) bool) {
    return func(yield func(int) bool) {
        for i := 1; i <= n; i++ {
            if !yield(i) {   // 呼び出し側が break すると false が返る
                return
            }
        }
    }
}

func main() {
    for v := range Count(3) {  // 1.23 から関数を range できる
        fmt.Println(v)         // 1 2 3
    }
}
```

- 標準ライブラリに `iter` パッケージ（`iter.Seq[T]` / `iter.Seq2[K,V]`）が追加され、`slices`/`maps` にイテレータ版（`slices.All` `maps.Keys` 等）が増えた。

```go
import ("maps"; "slices")
m := map[string]int{"a": 1, "b": 2}
keys := slices.Sorted(maps.Keys(m)) // イテレータ → ソート済みスライス
```

### その他 1.23 のトピック
- `time.Timer` / `time.Ticker` の GC 挙動とチャネルバッファが改善（停止後の誤発火が減る）。
- `unique` パッケージ追加（値のインターン＝重複排除で比較を高速化）。
- ジェネリクスはこの世代で実用的に成熟し、ライブラリでも一般的に使われる。

## ハマりどころ
- 未使用変数・未使用 import はエラー。デバッグ中の一時変数は `_ = x` で握りつぶす。
- `nil` スライスへの `append` は OK だが、`nil` マップへの書き込みは panic（`make` で初期化が必要）。
- ループ変数の取り回し: 1.22 以降は各反復で変数が新しく作られるよう変更されたため、goroutine 内で旧来必要だった `i := i` の再代入は不要になった（1.23 でも同じ）。
- インターフェースに `nil` の具体値を入れると、interface 自体は `nil` ではなくなる（`err != nil` 判定の典型的な罠）。
- エラーは戻り値で返すのが原則。`panic` を例外代わりに使わない。

## 関連
- フレームワーク: [../gin/](../gin/)（Gin・軽量 Web フレームワーク）
- 言語概要: [../README.md](../README.md)
