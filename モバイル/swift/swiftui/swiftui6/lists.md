# リスト（List / ForEach / Identifiable）（SwiftUI）

## ひとことで言うと
`List` は **縦スクロールする行の集合（テーブル）** を表す View。中身を**動的に**並べるとき `ForEach` を組み合わせ、各要素を一意に識別するために **`Identifiable`（または `id:`）** を使う。

## 役割・なぜ必要か
- 「配列を画面に並べる」という最頻出のUI（一覧・設定画面・チャットなど）を、少ない記述で実現するためにある。
- SwiftUI は宣言的なので、行を手で足し引きしない。**データ配列を変えれば List が自動で追従**する。差分更新のために「どの行がどの要素か」を SwiftUI が知る必要があり、そこで `id`（一意キー）が要る。
- `onDelete` / `onMove` を付けるだけで、スワイプ削除・並べ替えといった iOS 標準の編集操作が手に入る。

## 基本の書き方（コード）
```swift
struct Fruit: Identifiable {
    let id = UUID()   // ← 一意キー（Identifiable の必須要件）
    var name: String
}

struct FruitListView: View {
    @State private var fruits = [
        Fruit(name: "りんご"),
        Fruit(name: "みかん"),
        Fruit(name: "ぶどう"),
    ]

    var body: some View {
        List {
            // Section で見出し付きグループにできる
            Section("くだもの") {
                ForEach(fruits) { fruit in   // Identifiable なので id 指定不要
                    Text(fruit.name)
                }
                .onDelete { indexSet in
                    fruits.remove(atOffsets: indexSet)   // スワイプ削除
                }
                .onMove { from, to in
                    fruits.move(fromOffsets: from, toOffset: to)  // 並べ替え
                }
            }
        }
        .toolbar { EditButton() }  // 編集モードの ON/OFF（onMove に必要）
    }
}
```

## 実務での使い方・定番パターン
- **静的な行**（固定メニュー等）は `ForEach` 不要。`List { Text("A"); Text("B") }` でよい。
- **動的な行**は `ForEach(配列)`。配列要素が `Identifiable` なら `id:` を省略できる。
- **Identifiable にできない既存型**（数値配列など）は `ForEach(items, id: \.self)` のように `id:` を明示する。
- **行タップで遷移**は `NavigationStack` 内で `NavigationLink` を行に置く。→ [navigation.md](./navigation.md)
- **Section** で見出し・フッターを付けてグルーピング。設定画面風UIの定番。
- **スタイル**は `.listStyle(.insetGrouped)` 等で切り替え。
- 大量データでも `List` は内部で**遅延生成**するため、表示範囲の行だけ作られ軽い。

## ハマりどころ / アンチパターン
- **`id` 不在エラー**：`ForEach` に渡す要素が `Identifiable` でなく `id:` も無いとコンパイルできない。`Identifiable` 準拠か `id:` 明示のどちらかが**必須**。
- **`id: \.self` の落とし穴**：`String` や `Int` を `id: \.self` で並べると、**値が重複したとき**に行が壊れる（削除位置ズレ・アニメ崩れ）。重複し得るデータは一意な `id` プロパティを持たせる。
- **`id` に毎回 `UUID()` を計算プロパティで返す**のはNG。`body` 再評価のたびに id が変わり、差分更新が効かず全行が作り直しになる。`id` は**保存された値**にする（`let id = UUID()`）。
- **`onMove` が効かない**：編集モードに入っていないと動かない。`EditButton()` を置くか `.environment(\.editMode, ...)` を設定する。
- **`onDelete` をView全体でなく `ForEach` に付ける**点に注意（`List` ではなく `ForEach` のモディファイア）。
- 行の中で重い計算をすると、スクロール時に毎フレーム走って重くなる。整形は事前に済ませる。→ [views.md](./views.md)

## 関連
[views.md](./views.md) / [navigation.md](./navigation.md) / [state.md](./state.md)
