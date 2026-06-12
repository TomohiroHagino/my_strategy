# レイアウト（SwiftUI）

## ひとことで言うと
View を**縦・横・奥行き**に並べる仕組み。基本は **`VStack`（縦）・`HStack`（横）・`ZStack`（重ね）** の3つのコンテナで、`Spacer()` や `.frame()` `.padding()` でサイズと余白を調整する。

## 役割・なぜ必要か
- SwiftUI のレイアウトは「**親が子に提案 → 子が自分のサイズを決める**」という協調で決まる。座標を直接指定する代わりに、スタックに**並べる**ことで配置を宣言する。
- `VStack` / `HStack` / `ZStack` を**入れ子**にすれば、ほとんどの画面構成は座標計算なしで組める。レスポンシブ（画面サイズ違い）にも自然に追従する。
- `Spacer()` で「余白を押し広げる」、`.frame()` で「サイズを指定/制約する」、`alignment` で「揃え方」を決める。この3つで配置を細かく制御できる。

## 基本の書き方（コード）
```swift
import SwiftUI

struct LayoutSample: View {
    var body: some View {
        VStack(alignment: .leading, spacing: 16) {   // 縦並び・左揃え
            Text("タイトル").font(.title)

            HStack {                                  // 横並び
                Image(systemName: "star.fill")
                Text("評価 4.5")
                Spacer()                              // 残りの横幅を押し広げ右寄せ
                Text("詳細")
            }

            ZStack(alignment: .bottomTrailing) {      // 重ね合わせ
                Color.blue.opacity(0.2)
                Text("バッジ").padding(6)
            }
            .frame(height: 80)                        // 高さを指定
        }
        .padding()                                    // 全体に余白
        .frame(maxWidth: .infinity, alignment: .leading) // 横いっぱいに広げる
    }
}

#Preview { LayoutSample() }
```

## 実務での使い方・定番パターン
- **3スタックの使い分け**：縦に積むなら `VStack`、横に並べるなら `HStack`、画像の上に文字を重ねるなど奥行きは `ZStack`。これらを**入れ子**にして画面を構成するのが基本。
- **`Spacer()` で寄せる**：`HStack { Text("左"); Spacer(); Text("右") }` で両端寄せ。`Spacer()` は「使えるだけ伸びる空白」で、左右/上下の寄せに多用する。
- **`.frame(maxWidth: .infinity)`**：「使える幅いっぱいに広げる」定番。`alignment:` と組み合わせると、広げた領域の中での寄せも指定できる。カードを画面幅に合わせるときに頻出。
- **`spacing:` と `.padding()` の役割分担**：要素**間**の間隔は `VStack(spacing:)`、要素の**外周**余白は `.padding()`。混同しないと余白設計が安定する。→ [modifiers.md](./modifiers.md)
- **`GeometryReader` は最後の手段**：親のサイズを取って計算したいときに使う。ただし扱いが難しく、レイアウトを壊しやすいので、スタック＋`frame` で組めるならそちらを優先。

## ハマりどころ / アンチパターン
- **スタックの過剰ネスト**：`VStack` の中に意味なく `HStack` を何重にも入れると、見通しが悪くデバッグしづらい。深くなったら**子 View に切り出す**。→ [views.md](./views.md)
- **`.frame` の制約を誤解**：`.frame(width:height:)` は「希望サイズ」であって、中身がはみ出すこともある。固定値を入れすぎると画面サイズ違いで崩れる。可変にしたいときは `maxWidth/maxHeight` を使う。
- **`alignment` 指定漏れ**：`VStack` は既定で**中央揃え**。左揃えにしたいのに `alignment: .leading` を付け忘れて「なぜか中央に寄る」が頻出。スタックの `alignment` と `.frame` の `alignment` は**別物**なので両方意識する。
- **`Spacer()` を ZStack で使う**：`Spacer` は VStack/HStack の押し広げ用。`ZStack` では意図通りに効かないことが多い。
- **`GeometryReader` を安易に使う**：親の全領域を占有してしまい、周囲のレイアウトが崩れがち。本当に必要な範囲だけに限定する。

## 関連
[modifiers.md](./modifiers.md)
