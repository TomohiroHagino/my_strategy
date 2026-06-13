# Swift 6（言語解説）

## ひとことで言うと
Apple の静的型付け・安全志向の言語で、値型（struct / enum）中心・Optional による null 安全・プロトコル指向が特徴。Swift 6 の目玉はコンパイル時のデータ競合検出（data race safety）で、`async`/`await` と actor を使った並行コードの安全性を型システムで保証するようになった。

## このバージョンの位置づけ（リリース / サポート / どこで使うか）
- Swift 6 は 2024 年後半リリース。最大の変更は完全な並行安全チェックの導入。Swift 6 言語モードを有効にすると、データ競合になりうるコードがコンパイルエラーになる。
- 互換性のため、既存コードは Swift 5 言語モードのままビルドでき、6 への移行は段階的に行える（警告として先に出してから対応できる）。
- 主用途は iOS / macOS / watchOS / visionOS のネイティブアプリ。Linux / Windows やサーバ（Vapor 等）でも動くが、中心は Apple プラットフォーム。開発には Mac と Xcode が必要。

```bash
swift --version
swift package init --type executable   # SwiftPM プロジェクト雛形
swift build
swift run
```

## 言語の基本（文法の要点）
変数は `let`（再代入不可）と `var`（可変）。型推論が効くが明示もできる。

```swift
let name = "Swift"       // 推論で String、再代入不可
var count = 0            // 可変
let pi: Double = 3.14    // 明示型
```

関数は `func`。引数ラベルが呼び出し側に現れるのが特徴。`_` でラベルを省ける。

```swift
func greet(_ name: String, with greeting: String = "Hello") -> String {
    "\(greeting), \(name)!"
}

greet("A")                    // 第1引数はラベルなし
greet("B", with: "Hi")        // with は外部ラベル
```

`if` / `guard`（早期脱出）/ `switch`（網羅必須）が制御構文。`switch` は全ケースを書くか `default` が要る。

```swift
func describe(_ n: Int) -> String {
    guard n >= 0 else { return "negative" }   // 早期 return
    switch n {
    case 0: return "zero"
    case 1...9: return "small"
    default: return "large"
    }
}
```

文字列補間 `\(式)`、範囲 `1...10` / `1..<10`、`for x in seq` などが基本。

## この言語の核心概念（他言語と違う・必ず押さえる）
Swift は「値型・Optional・ARC・プロトコル指向」が背骨。文法の前に、この6点を押さえる。

### 値型中心（struct / enum）vs 参照型（class）
①何か：`struct` と `enum` は値型で、代入や関数渡しでコピーされる。`class` は参照型で共有される。Swift は標準ライブラリの `Array`/`String`/`Int` まで含め値型を第一に設計され、値型のメソッドが自身を変える場合は `mutating` を付ける。

②具体コード：
```swift
struct Point { var x: Int; var y: Int
    mutating func moveRight() { x += 1 }   // 値型を変えるには mutating
}
var a = Point(x: 1, y: 2)
var b = a          // コピー
b.x = 99
print(a.x)         // 1（a は影響を受けない）
```

③他言語と違う点/つまずき：Java/Python では基本的に全てが参照（オブジェクトは共有）で、コピーは明示操作。Swift は逆で、struct は渡すたびにコピーのセマンティクス。意図せず「コピーされて変更が伝わらない」「class だと思って共有されてしまう」取り違えが起きる。`let` で束ねた struct はプロパティも変更不可になる（`mutating` も呼べない）。

### Optional（必修の null 安全）
①何か：「値があるかもしれない」を `T?`（`Optional<T>`）で表す。中身を取り出すには `if let`/`guard let` で安全に開くか、`??` で代替を与える。`!` は強制アンラップで、nil なら即クラッシュ。

②具体コード：
```swift
var maybe: String? = nil
let len = maybe?.count ?? 0       // nil なら 0
if let value = maybe {            // 取り出せたときだけ実行
    print(value)
}
func use(_ s: String?) {
    guard let s else { return }   // nil なら早期脱出、以降 s は非 Optional
    print(s.count)
}
```

③他言語と違う点/つまずき：Optional は単なる「null 許容フラグ」ではなく、`some(値)` / `none` を持つ列挙型。だから `maybe?.count` の結果も `Int?` になり「Optional のチェーン」が起きる。`if let`/`guard let` で開かないと中身を使えない点が、null をそのまま参照できる言語と違う。`!` の常用は Optional を骨抜きにする。

### ARC（自動参照カウント）と循環参照 / `weak`
①何か：Swift はクラスインスタンスの参照数を数え、0 になった瞬間に解放する（ARC）。互いに強参照し合うと参照数が 0 にならずメモリリークするので、片方を `weak`（弱参照、自動で nil 化）か `unowned` にして断ち切る。

②具体コード：
```swift
class Node {
    var next: Node?
    weak var parent: Node?   // 親子で循環しないよう親側を弱参照に
}
class VM {
    var onTap: (() -> Void)?
    func bind() { onTap = { [weak self] in self?.handle() } }  // クロージャの循環も切る
    func handle() {}
}
```

③他言語と違う点/つまずき：Java/Go/Kotlin はトレースする GC が循環参照ごと回収するので、開発者は循環を気にしない。Swift の ARC は循環を自動では壊せないため、設計時に「どちらの参照を弱くするか」を人間が決める。特にクロージャが `self` を強キャプチャしてリークするのが定番の罠で、`[weak self]` で切る。

### プロトコル指向（protocol + extension のデフォルト実装）
①何か：振る舞いを `protocol` で定義し、`extension` でデフォルト実装を与える。継承の縦の階層ではなく、複数のプロトコル準拠を合成して設計する。値型（struct/enum）も同じ仕組みに乗れる。

②具体コード：
```swift
protocol Greetable { var name: String { get } }
extension Greetable {
    func greet() -> String { "Hello, \(name)!" }  // デフォルト実装
}
struct Person: Greetable { let name: String }
Person(name: "A").greet()        // 何も書かずに使える
```

③他言語と違う点/つまずき：Java のインターフェースは（default メソッドを除き）実装を持てず、単一継承の制約がある。Swift は protocol + extension で実装を共有でき、class を介さず struct/enum でもポリモーフィズムを得る。「is-a の継承」より「can-do の準拠」で考える発想の転換が要る。`some`（不透明型）/ `any`（存在型）の使い分けも最初に迷う。

### async/await + actor（Swift 6 のデータ競合安全）
①何か：非同期は `async`/`await` で中断点を明示して書く。`actor` は内部状態へのアクセスを 1 つずつに直列化してデータ競合を防ぐ型。Swift 6 言語モードでは、スレッドをまたいで渡す値に `Sendable` 準拠を要求し、満たさない受け渡しをコンパイルエラーにする。

②具体コード：
```swift
actor Counter {
    private var value = 0
    func increment() { value += 1 }   // 同時アクセスでも安全
}
let c = Counter()
await c.increment()                   // actor 越しなので await

@MainActor
func updateUI(_ text: String) { /* 必ずメインスレッドで実行 */ }
```

③他言語と違う点/つまずき：多くの言語ではデータ競合は実行時に初めて顕在化する（あるいは気づかない）。Swift 6 は型システムで競合の可能性をコンパイル時に弾く。既存コードを 6 モードにすると `Sendable` 違反や actor 隔離違反で大量のエラーが出るので、段階移行する。UI 更新は `@MainActor` で固定しないと非メインスレッド呼び出しの警告になる。`Task { }` 単発は構造化されず、キャンセル伝播を自分で管理する必要がある。

### `guard` とパターンマッチ（`switch` 網羅）
①何か：`guard 条件 else { 脱出 }` は「条件を満たさなければ早期脱出」する制御構文で、満たした後は本流をネストせず書ける。`switch` は全ケースを網羅するか `default` が必須で、enum の associated value をパターンで取り出せる。

②具体コード：
```swift
enum Result { case success(Int), failure(String) }
func handle(_ r: Result) -> Int {
    switch r {
    case .success(let v): return v      // 値を取り出す
    case .failure: return -1
    // enum を網羅すれば default 不要
    }
}
```

③他言語と違う点/つまずき：`guard` は `if` の逆で「正常系を本流に残しネストを浅くする」ための専用構文。`guard let` で開いた値は脱出後のスコープでも生きる（`if let` はブロック内だけ）点が独特。`switch` は C 系と違いフォールスルーしない（暗黙の `break`）。enum を網羅していれば `default` 不要で、ケース追加漏れをコンパイラが教えてくれる。

## 型・データモデル
- 基本型は `Int` / `Double` / `Bool` / `String` / `Character`。コレクションは `Array` / `Dictionary` / `Set`。これらはすべて値型。
- 値型（struct / enum）はコピーで渡る。参照型（class）は共有される。Swift は値型を第一に設計されている。

```swift
struct Point { var x: Int; var y: Int }
var a = Point(x: 1, y: 2)
var b = a            // コピー
b.x = 99
print(a.x)           // 1（a は影響を受けない）
```

Optional が null 安全の核。`?` 付きが「値があるかもしれない」型で、`if let` / `guard let` / `??` で安全に取り出す。

```swift
var maybe: String? = nil
let len = maybe?.count ?? 0       // ?. と ?? デフォルト
if let value = maybe {            // Optional バインディング
    print(value)
}
let forced = maybe!               // ! は値があると断言（nil ならクラッシュ）
```

enum は値を持てる（associated values）。状態の表現に強い。

```swift
enum Result {
    case success(Int)
    case failure(String)
}

func handle(_ r: Result) -> Int {
    switch r {
    case .success(let v): return v      // 値を取り出す
    case .failure: return -1
    }
}
```

## この言語らしさ / 特徴的な機能
プロトコル指向。振る舞いをプロトコルで定義し、extension でデフォルト実装を与える。継承ではなく合成で設計する。

```swift
protocol Greetable {
    var name: String { get }
}
extension Greetable {
    func greet() -> String { "Hello, \(name)!" }  // デフォルト実装
}

struct Person: Greetable { let name: String }
Person(name: "A").greet()
```

extension は既存の型（自作・標準ライブラリ問わず）にメソッドや計算プロパティを足せる。

```swift
extension Int {
    var isEven: Bool { self % 2 == 0 }
}
4.isEven   // true
```

ジェネリクスと `some`（不透明型）/ `any`（存在型）でプロトコルを型として扱う。クロージャ、`mutating`（値型のメソッドが自身を変更）、プロパティラッパなども特徴。

メモリ管理は ARC（自動参照カウント）。参照型の循環参照は `weak` / `unowned` で断ち切る。

```swift
class Node {
    var next: Node?
    weak var parent: Node?   // 循環参照を避けるため weak
}
```

## 並行・非同期
非同期は `async`/`await`。中断点が `await` で明示される。

```swift
func fetch() async throws -> String {
    try await Task.sleep(for: .seconds(1))
    return "data"
}
```

並行の起点は `Task`、複数の並行子は `async let` や TaskGroup で扱い、`withTaskGroup` などは構造化される。

```swift
func loadBoth() async throws -> (String, String) {
    async let a = fetch()       // 並行起動
    async let b = fetch()
    return try await (a, b)     // 両方そろうまで待つ
}
```

actor は内部状態を1つずつのアクセスに直列化し、データ競合を防ぐ。actor の状態に外から触るには `await` が要る。

```swift
actor Counter {
    private var value = 0
    func increment() { value += 1 }   // 同時アクセスでも安全
    func current() -> Int { value }
}

let c = Counter()
await c.increment()    // actor 越しなので await
```

Swift 6 の data race safety では、スレッドをまたいで渡せる型に `Sendable` 準拠を要求し、満たさない値の受け渡しをコンパイルエラーにする。UI を扱うコードは `@MainActor` でメインスレッドに固定する。

```swift
@MainActor
func updateUI(_ text: String) { /* 必ずメインスレッドで実行 */ }
```

## 標準ライブラリ / ツールチェーン
- 標準ライブラリにコレクションとその高階関数（`map` / `filter` / `reduce`）、`Optional`、`Result`、文字列処理、`Codable`（JSON 等の直列化）が揃う。Foundation で日付・ファイル・ネットワーク等を補う。
- 依存管理は Swift Package Manager（SPM）。`Package.swift` に依存を書く。CocoaPods / Carthage もまだ使われる。
- ビルド・テスト・デバッグは Xcode が中心。テストは XCTest と、新しい Swift Testing（マクロベースの `@Test`）。

```swift
// Package.swift（抜粋）
dependencies: [
    .package(url: "https://github.com/apple/swift-algorithms", from: "1.2.0")
]
```

## このバージョンの新機能・トピック
- 完全並行チェック（data race safety）：Swift 6 言語モードで、データ競合の可能性をコンパイル時に検出。`Sendable`・actor 隔離・`@MainActor` を型レベルで強制する。これが 6 の最大の主題。
- 段階移行：プロジェクトやモジュール単位で Swift 5 / 6 言語モードを選べ、まず警告で洗い出してから対応できる。
- typed throws：投げるエラーの型を関数シグネチャで指定できるようになった。

```swift
enum FileError: Error { case notFound }
func read() throws(FileError) -> String { throw FileError.notFound }
```

- 所有権関連の `borrowing` / `consuming` 引数指定や、`count(where:)` などの標準ライブラリ追加も入っている。Embedded Swift（組み込み向けサブセット）も進展。

## ハマりどころ
- `!`（強制アンラップ）：nil を `!` で開くとクラッシュ。`if let` / `guard let` / `??` を基本にする。
- 値型と参照型の取り違え：struct はコピー、class は共有。意図せず共有・コピーされて状態が食い違う。
- ARC の循環参照：互いに strong 参照し合うとメモリリーク。クロージャの `self` キャプチャも `[weak self]` で切る。
- Swift 6 並行への移行：既存コードが `Sendable` 違反や actor 隔離違反で大量のエラー・警告を出す。一度に直さず段階的に。
- `@MainActor` 外し忘れ：UI 更新が非メインスレッドから呼ばれてランタイム警告になる。
- `Task` の独立性：単に `Task { }` で起動したものは構造化されず、キャンセル伝播やライフサイクル管理を自分で気にする必要がある。

## 関連
- [../swiftui/](../swiftui/) … Apple の現行宣言的 UI フレームワーク SwiftUI の版別リファレンス。
- [../xcode.md](../xcode.md) … iOS 開発に必須の IDE「Xcode」の解説。
- 同フォルダの [README.md](../README.md) … Swift 言語の概要・強み弱み・エコシステム。
