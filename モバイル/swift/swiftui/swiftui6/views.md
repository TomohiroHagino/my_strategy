# View（SwiftUI）

## ひとことで言うと
画面に表示する「部品」を表す型。`View` プロトコルに準拠した **`struct`（値型）** で、`body` プロパティに「どう見えるか」を宣言的に書く。小さな View を**合成**して大きな画面を組み立てる。

## 役割・なぜ必要か
- SwiftUI では「画面そのもの」も「ボタン1つ」も、すべて **View**。`Text` / `Image` / `Button` といった部品も、それを組み合わせた自作の画面も、同じ `View` という単位で扱える。
- View は**軽量な値型（struct）**。状態が変わるたびに何度も作り直される前提で設計されており、「手で画面を書き換える」のではなく「**状態を変えると body が再評価されて画面が追従する**」という宣言的UIの中心を担う。
- 大きな1枚岩の View ではなく、**小さな View に分割して合成**することで、再利用・可読性・プレビューのしやすさが上がる。

## 基本の書き方（コード）
```swift
import SwiftUI

// View プロトコルに準拠した struct。body に見た目を宣言する。
struct GreetingView: View {
    let name: String

    // some View = 「View の一種を返す」（具体型は隠す＝opaque type）
    var body: some View {
        Text("Hello, \(name)!")
            .font(.title)
            .foregroundStyle(.blue)
    }
}

// 小さな View を合成して画面を組む
struct ProfileCard: View {
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            GreetingView(name: "Taro")   // 自作 View を部品として使う
            Text("iOS Developer")
                .font(.subheadline)
                .foregroundStyle(.secondary)
        }
        .padding()
    }
}

#Preview {
    ProfileCard()
}
```

## 実務での使い方・定番パターン
- **小さく分割して合成**：1つの View が大きくなったら、意味のある単位（ヘッダー、行、カードなど）で `struct` に切り出す。`body` の中で `HeaderView()` のように呼ぶだけで部品化できる。
- **データは引数で渡す**：表示に必要な値は `let title: String` のようにプロパティで受け取る。`init` を自分で書かなくても、`struct` の自動生成イニシャライザで `GreetingView(name: "Taro")` と渡せる。
- **`some View` を返す**：`body` の型は基本 `some View`。返している具体的な型（`Text` か `VStack` か等）を**隠蔽**しつつ「View である」ことだけを約束する仕組み（opaque type）。型を毎回書かずに済む。
- **`@ViewBuilder` の恩恵**：`body` や `VStack { ... }` の中身は、複数の View を**並べて書くだけ**で組み合わせられる。これは裏で `@ViewBuilder` が効いているため。→ [layout.md](./layout.md)
- **修飾は modifier で**：色・余白・フォントなどの見た目は `.padding()` などの modifier で付ける。→ [modifiers.md](./modifiers.md)

## ハマりどころ / アンチパターン
- **`body` はビュー1つにまとめる**：`body` が返せるのは原則「1つの View」。複数の要素を並べたいときは `VStack` などの**コンテナで包む**。直に2つ並べるとエラー（厳密には `@ViewBuilder` が許す範囲を超えると崩れる）。
- **`some View` の意味を取り違える**：`some View` は「**何か1種類の View**」。`if` 分岐で型が違う View を返す場合などは `@ViewBuilder` が吸収してくれるが、戻り値の型を自分で固定したい場面では `AnyView` を使うこともある（多用は避ける、性能・可読性が落ちる）。
- **View を値型（struct）として扱わない**：View は struct（値型）。**手元のインスタンスを書き換えて画面を更新しようとしても無駄**。更新は「状態を変える」ことで行う。→ [state.md](./state.md)
- **巨大な `body`**：1つの `body` に何百行も詰め込むと再利用もプレビューも辛い。早めに小さい View へ分割する。
- **重い計算を `body` に直書き**：`body` は何度も再評価される。重い処理は `body` の外（プロパティや関数）に出す。

## 関連
[layout.md](./layout.md) / [modifiers.md](./modifiers.md)
