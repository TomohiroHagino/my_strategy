# modifier（.padding() 等・適用順序）（SwiftUI）

## ひとことで言うと
View に `.padding()`・`.background(.blue)` のようにドットで連ねて見た目や振る舞いを足す仕組み。各 modifier は **「元の View を包んだ新しい View を返す」**。元の View を書き換えるのではなく、ラップして新しい View を作るのがポイント。

## 役割・なぜ必要か
- 「余白・背景・枠・角丸・影」などを宣言的に積み上げて UI を組み立てるため。
- View 本体（`Text` など）は単機能に保ち、装飾は modifier の合成で表現する → 再利用しやすく読みやすい。
- modifier は**新しい View を返す**ので、`.padding().background(...)` の順番がそのまま「外側へ包んでいく順番」になる。**順序＝レイアウト結果**になるのがSwiftUIの特徴。

## 基本の書き方（コード）
```swift
import SwiftUI

struct BadgeView: View {
    var body: some View {
        // 内側 → 外側へ。上から順に「包んで」いく
        Text("NEW")
            .font(.caption.bold())
            .padding(.horizontal, 12)        // 文字の周りに余白
            .padding(.vertical, 6)
            .background(.blue)               // 余白込みの矩形を青で塗る
            .foregroundStyle(.white)
            .clipShape(Capsule())            // カプセル型に切り抜く
            .shadow(radius: 4)               // 切り抜いた形に影
    }
}
```

```swift
// 順序で結果が変わる代表例：padding と background
struct OrderDemo: View {
    var body: some View {
        VStack(spacing: 16) {
            // A: padding が先 → 背景は「余白を含んだ範囲」を塗る
            Text("padding → background")
                .padding()
                .background(.yellow)

            // B: background が先 → 背景は「文字ぴったり」、余白は外側に付く
            Text("background → padding")
                .background(.yellow)
                .padding()
        }
    }
}
```

```swift
// カスタム ViewModifier：装飾の組み合わせに名前を付けて再利用
struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.white)
            .clipShape(RoundedRectangle(cornerRadius: 12))
            .shadow(color: .black.opacity(0.1), radius: 8, y: 4)
    }
}

extension View {
    func cardStyle() -> some View { modifier(CardStyle()) }
}

struct CardDemo: View {
    var body: some View {
        Text("カード").cardStyle()       // 呼び出しは1行で済む
    }
}
```

## 実務での使い方・定番パターン
- **内側→外側の順で読む**：上に書いた modifier が内側、下が外側。`.frame` → `.background` → `.padding` の並びで「枠を決める→塗る→外側に余白」と意図を作る。
- **`clipShape` / `RoundedRectangle` で角丸**：`.background(...).clipShape(RoundedRectangle(cornerRadius:))` が定番。背景を付けてから切り抜く。
- **`overlay` / `background` で枠線**：`.overlay(RoundedRectangle(cornerRadius: 12).stroke(.gray))` で角丸枠。
- **カスタム `ViewModifier` ＋ `extension View`**：3つ以上の装飾の組み合わせはまとめて `.cardStyle()` のような1メソッドにする。→ [views.md](./views.md)
- **共通スタイルは1か所に**：色・角丸・余白はカスタム modifier や定数に寄せて、画面ごとのブレを防ぐ。→ [layout.md](./layout.md)

## ハマりどころ / アンチパターン
- **順序の取り違えが最頻ミス**：`padding → background` と `background → padding` は見た目が別物。「背景が余白まで広がらない／逆に広がりすぎ」はほぼ順序の問題。
- **`frame` を付ける位置で大きさが変わる**：`.frame(maxWidth: .infinity)` を `.background` の前後どちらに置くかで「塗り範囲」も変わる。
- **`clipShape` の後に `shadow`**：切り抜き後に影を付けないと、影まで一緒にクリップされて消える／不自然になる。影は最後の方に。
- **modifier の返り値を捨てる発想**：modifier は元 View を変更せず**新しい View を返す**ので、`view.padding()` の戻りを使わないと何も起きない（チェーンの途中で値を握り潰さない）。
- **何でもチェーンして長大化**：10行を超える装飾はカスタム `ViewModifier` に切り出す。可読性と再利用性が上がる。
- **`if` 条件で modifier を分岐しすぎる**：型が変わって複雑化しがち。条件付きスタイルは `ViewModifier` 側や `.background` の中で吸収する方が素直。

## 関連
[views.md](./views.md) / [layout.md](./layout.md)
