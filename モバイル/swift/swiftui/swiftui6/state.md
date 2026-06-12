# 状態（@State / @Binding / @Observable）（SwiftUI）

## ひとことで言うと
View が持つ「変わりうる値」のこと。SwiftUI は **状態が変わると、その状態を使っている View の `body` を再評価して画面を描き直す**。`@State`・`@Binding`・`@Observable`（iOS17+）などのプロパティラッパで「これは状態だ」と宣言する。

## 役割・なぜ必要か
- View は `struct`（値型）で、手で `label.text = ...` のように書き換えられない。代わりに **状態を変える → SwiftUI が差分を描く** という流れにする。
- `@State` … その View **ローカル**の単純な状態（カウンタ、トグル、選択中のタブ等）。View が所有者。
- `@Binding` … 親が持つ状態への**参照**。子に「読み書き両方できる窓口」を渡す（双方向）。
- `@Observable` クラス（iOS17+）… 複数プロパティを持つ**参照型の状態**。画面間で共有したり、ロジックを切り出したいとき。
- 「どこが所有者で、どう共有するか」を型で表すのが SwiftUI の状態管理の肝。

## 基本の書き方（コード）
```swift
import SwiftUI

// 1) @State：View ローカルの状態。必ず private で持つ
struct CounterView: View {
    @State private var count = 0          // この View が所有する

    var body: some View {
        VStack {
            Text("count: \(count)")
            Button("+1") { count += 1 }    // 状態を変える → body 再評価
            StepperRow(value: $count)      // $ で Binding を子へ渡す
        }
    }
}

// 2) @Binding：親の状態への双方向の窓口
struct StepperRow: View {
    @Binding var value: Int                // 親の count と繋がる

    var body: some View {
        HStack {
            Button("-") { value -= 1 }     // 親側の値が変わる
            Button("+") { value += 1 }
        }
    }
}

// 3) iOS17+ の @Observable クラス：参照型の状態
@Observable
final class CartModel {
    var items: [String] = []
    var total: Int = 0
    func add(_ name: String) {             // メソッドで状態を変える
        items.append(name)
        total += 1
    }
}

struct CartView: View {
    @State private var cart = CartModel()  // @Observable は @State で保持

    var body: some View {
        VStack {
            Text("点数: \(cart.total)")
            Button("追加") { cart.add("apple") }
            DetailRow(cart: cart)          // 参照型なのでそのまま渡せる
        }
    }
}

struct DetailRow: View {
    @Bindable var cart: CartModel          // $cart.xxx で Binding が欲しい時
    var body: some View {
        Text(cart.items.last ?? "なし")
    }
}
```

## 実務での使い方・定番パターン
- **`@State` は private**：その View が唯一の所有者であることを表す。外から初期値を渡したいだけなら通常の `let` プロパティにする。
- **`$value` で Binding を作る**：`TextField("名前", text: $name)` のように `$` 接頭辞で `@State` を `Binding` に変換して子へ渡す。
- **iOS17 以降は `@Observable` が基本**：クラスに `@Observable` を付け、View 側は `@State private var model = Model()` で保持。共有したいときは下位 View へそのまま渡すか `@Environment` に入れる。→ [data_flow.md](./data_flow.md)
- **`@Bindable`**：`@Observable` のプロパティから `$model.name` のような Binding が欲しいとき（フォーム等）に使う。→ [forms_input.md](./forms_input.md)
- **使い分けの目安**：単一の値で View ローカル → `@State`／親子で共有 → `@Binding`／複数プロパティ・ロジック・画面共有 → `@Observable`。

## ハマりどころ / アンチパターン
- **`@State` を public にする / 外から初期化して上書き**：所有者が曖昧になり再描画が壊れる。`@State` は private で、初期値は宣言時に与えるのが原則。
- **`$` を付け忘れる / 付けすぎる**：子が `@Binding` を要求しているのに `value`（値）を渡すとコンパイルエラー。逆に値で良い所に `$` を付けると型不一致。
- **iOS17 の `@Observable` と旧 `ObservableObject` の混同**：
  - 旧来は `class M: ObservableObject { @Published var x }` ＋ View 側 `@StateObject`／`@ObservedObject`。
  - iOS17+ は `@Observable class M { var x }` ＋ View 側 `@State`／`@Bindable`。`@Published` は不要。
  - 旧 API は **使ったプロパティだけでなくオブジェクト全体**で再描画が起きやすいが、`@Observable` は**実際に読んだプロパティ単位**で追跡され無駄な再描画が減る。
- **`@StateObject` を子で `@StateObject` として再生成**：所有は1か所（生成元）だけ。子は `@ObservedObject`（旧）や素の受け取りにする。
- **構造体を `@Observable` にしようとする**：`@Observable` はクラス専用。値型の共有は `@Binding` を使う。
- **`body` 内で状態を直接書き換える**：描画ループ中の変更は警告・無限ループの原因。変更はボタンアクションや `.task` などイベント側で行う。

## 関連
[data_flow.md](./data_flow.md) / [forms_input.md](./forms_input.md)
