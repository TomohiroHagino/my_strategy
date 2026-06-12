# ViewInspector（SwiftUI）

## ひとことで言うと
SwiftUI の **View を単体テストする**サードパーティ製ライブラリ。`view.inspect()` で View の**内部階層（VStack→Text など）を実行時に走査**し、テキスト・状態・アクションを検証する。標準の XCTest だけでは SwiftUI の `body` の中身を読めない（描画されないと中身が確認できない）ため、その隙間を埋める。UI 全体を動かす XCUITest（E2E）とは別レイヤーで、シミュレータ起動なしの高速テスト。

## 役割・なぜ必要か
- SwiftUI の View は `struct` で `body` が宣言的なため、**XCTest からは「中に何が描かれるか」を直接検査できない**。ViewInspector がその階層アクセスを提供する。
- シミュレータでアプリ全体を起動する XCUITest は遅く不安定。**View 1個だけをメモリ上で検査**して、表示や分岐（条件で出る/出ない）を高速に守れる。
- ボタンの `tap()` を擬似的に呼んで `@State` が変わり表示が切り替わることを、**実機操作なしで検証**できる。
- E2E（XCUITest）はテストピラミッドの頂点として重要フローに少数。ViewInspector は土台側を厚くする。

## 基本の書き方（コード）
```swift
// Swift Package Manager（テストターゲットに追加）
// .package(url: "https://github.com/nalexn/ViewInspector", from: "0.10.0")
```
```swift
// 対象 View
struct GreetingView: View {
    let name: String
    var body: some View {
        VStack {
            Text("Hello, \(name)")
            Text("Welcome")
        }
    }
}
```
```swift
// GreetingViewTests.swift
import XCTest
import ViewInspector
@testable import MyApp

final class GreetingViewTests: XCTestCase {
    func test_名前がテキストに表示される() throws {
        // Arrange
        let sut = GreetingView(name: "Taro")
        // Act: inspect() で階層を走査（VStack の0番目の Text）
        let text = try sut.inspect().vStack().text(0).string()
        // Assert
        XCTAssertEqual(text, "Hello, Taro")
    }

    func test_find_で文言から探す() throws {
        let sut = GreetingView(name: "Taro")
        // find: 階層パスを書かずに文字列で検索
        XCTAssertNoThrow(try sut.inspect().find(text: "Welcome"))
    }
}
```
```swift
// ボタンの tap で @State が変わり表示が変わる
struct CounterView: View {
    @State private var count = 0
    var body: some View {
        VStack {
            Text("count: \(count)")
            Button("+1") { count += 1 }
        }
    }
}

func test_ボタンでカウントが増える() throws {
    let sut = CounterView()
    let view = try sut.inspect()
    try view.vStack().button(1).tap()                 // ボタンを擬似タップ
    let label = try view.vStack().text(0).string()
    XCTAssertEqual(label, "count: 1")
}
```

## 実務での使い方・定番パターン
- **階層パス vs `find`**：`vStack().text(0)` のようにパスで辿る方法と、`find(text:)` / `find(ViewType.Button.self)` で**構造に依存せず探す**方法。リファクタ耐性は `find` 系が高い。
- **`@State` を伴う検証は `on(inspection)` パターン**：内部 `@State` の変化を見るには、View に `inspection` フックを仕込んで `view.on(\.didAppear)` で非同期に検査する公式パターンを使う（ライブラリの README 手順に従う）。
- **ViewModel と組み合わせ**：`@Observable`/`ObservableObject` の VM をテスト側で操作し、その状態が View のテキストへ反映されることを ViewInspector で確認する。→ [data_flow.md](./data_flow.md)
- **`callOnTapGesture()` / `button().tap()`**：ジェスチャやボタンのアクションを擬似的に発火し、状態遷移を検証。
- **modifier の検証**：`.foregroundColor` などは取得できる範囲が限られる。**色・装飾より「何が表示され、操作で何が変わるか」**に注力する（見た目の固定はスナップショットテストの領域）。
- **役割分担**：ロジック寄りは ViewInspector + XCTest、重要フローの実機通しは XCUITest（標準）。スナップショット（見た目固定）は別ライブラリ。

## ハマりどころ
- **階層パスのズレ**：`vStack().text(0)` は構造に密結合で、`if`/`Group`/暗黙コンテナが挟まると番号がずれて落ちる。`find(text:)` など検索系に切り替えると堅い。
- **`@State` を直接読めない**：View 値型のプロパティを外から覗くだけでは状態変化を追えない。`inspection` フック（`on(...)`）の公式セットアップが必要。これを飛ばすと「変わったはずが取れない」。
- **非同期の取りこぼし**：`.task`/`onAppear` での更新は即時に反映されない。`expectation` + `on(\.didAppear)` で**待ってから**検証する。
- **取得できない View 種別**：一部のシステム/プライベートなラッパーや複雑な generic は inspect できないことがある。対象を小さな View に切り出すと検査しやすい。
- **バージョン依存**：ViewInspector は SwiftUI 内部構造に依存するため、Xcode/SwiftUI 更新で API が壊れることがある。ライブラリのバージョンを iOS SDK に合わせて追従する。
- **見た目を検証し過ぎ**：色・余白の厳密検証は壊れやすい。レイアウト崩れの検出はスナップショットテストに任せ、ViewInspector は表示文言と操作結果に集中する。
- **E2E と混同**：実機権限ダイアログ・画面遷移の通しは XCUITest の担当。ViewInspector は View 単体の検査に留める。

## 関連
[views.md](./views.md) / [state.md](./state.md) / [data_flow.md](./data_flow.md) / [pitfalls.md](./pitfalls.md)
