# ナビゲーション（NavigationStack / sheet）（SwiftUI）

## ひとことで言うと
画面遷移の仕組み。iOS16+ の **`NavigationStack`** が基本で、`NavigationLink` で push（奥へ進む）、`.sheet` / `.fullScreenCover` でモーダル（下から／全画面で被せる）を出す。遷移先は **`.navigationDestination(for:)`** で値の型ごとに宣言する。

## 役割・なぜ必要か
- アプリの「画面の積み重ね（スタック）」を管理し、戻る・進むを成立させるため。
- `NavigationStack` は **`path` 配列で遷移状態を保持**できるので、ボタン操作だけでなく**プログラムから任意の画面へ飛ぶ・一気に戻る**が可能。ディープリンクや「保存後にトップへ戻る」等で必須。
- push（階層を進む）とモーダル（一時的に被せる）を **用途で使い分ける**ことで、ユーザーの文脈を壊さない遷移にできる。

## 基本の書き方（コード）
```swift
import SwiftUI

struct Item: Identifiable, Hashable {
    let id: Int
    let name: String
}

// 1) NavigationStack + NavigationLink + navigationDestination(for:)
struct ListScreen: View {
    let items = [Item(id: 1, name: "りんご"), Item(id: 2, name: "みかん")]

    var body: some View {
        NavigationStack {
            List(items) { item in
                NavigationLink(item.name, value: item)   // 値を渡して push
            }
            .navigationTitle("一覧")
            .navigationDestination(for: Item.self) { item in
                DetailScreen(item: item)                 // 型ごとに遷移先を宣言
            }
        }
    }
}

struct DetailScreen: View {
    let item: Item
    var body: some View { Text(item.name).font(.largeTitle) }
}
```

```swift
// 2) path でプログラム遷移（任意の画面へ飛ぶ / 一気に戻る）
struct RootScreen: View {
    @State private var path: [Item] = []          // スタックの状態

    var body: some View {
        NavigationStack(path: $path) {
            Button("2階層先へ") {
                path = [Item(id: 1, name: "A"), Item(id: 2, name: "B")]
            }
            .navigationDestination(for: Item.self) { item in
                VStack {
                    Text(item.name)
                    Button("ルートへ戻る") { path.removeAll() }  // 一気にトップ
                }
            }
        }
    }
}
```

```swift
// 3) sheet / fullScreenCover（モーダル）
struct ModalDemo: View {
    @State private var showSheet = false
    @State private var showCover = false

    var body: some View {
        VStack {
            Button("シート") { showSheet = true }
            Button("全画面") { showCover = true }
        }
        .sheet(isPresented: $showSheet) {            // 下からカード状に
            Text("シート").presentationDetents([.medium, .large])
        }
        .fullScreenCover(isPresented: $showCover) {  // 全画面で被せる
            VStack {
                Text("全画面")
                Button("閉じる") { showCover = false }
            }
        }
    }
}
```

## 実務での使い方・定番パターン
- **`value:` 付き `NavigationLink` ＋ `navigationDestination`**：遷移先を「渡す値の型」で集約宣言する現行スタイル。リンク側は値だけ渡し、画面の組み立ては destination 側に寄せる。
- **`@State private var path` でスタックを保持**：保存完了でトップへ戻す（`path.removeAll()`）、特定画面まで戻す（要素を削る）等をコードで制御。→ [state.md](./state.md)
- **`.sheet(item:)`** を使うと「どのデータでシートを開いたか」を `Identifiable` な値で渡せる（編集対象の受け渡しに便利）。
- **`presentationDetents`** で sheet の高さ（`.medium` / `.large` / `.height(...)`）を指定。半モーダルが定番。
- **使い分け**：階層を進む＝push（`NavigationLink`）、一時的な入力・確認＝`.sheet`、戻れない/没入させたい全画面＝`.fullScreenCover`。

## ハマりどころ / アンチパターン
- **旧 `NavigationView` を使い続ける**：iOS16+ では非推奨。`NavigationStack` に置き換える。`NavigationView` は挙動が読みにくく path 制御もできない。
- **`navigationDestination(for:)` の型ミスマッチ**：`NavigationLink(value: item)` の値の型と `navigationDestination(for: Item.self)` の型が一致しないと遷移しない（無反応）。`Hashable` 準拠も必須。
- **`navigationDestination` を List の行（ForEach内）に置く**：スタック直下に1回だけ置く。行ごとに置くと意図しない多重登録になる。
- **push と sheet の混同**：「奥へ進む」感覚は push、「一時的に被せて元へ戻る」は sheet。新規作成フォームを push にすると戻る導線が不自然になりがち。
- **`fullScreenCover` に閉じる手段が無い**：全画面は自動の下スワイプ閉じが効かないことがある。必ず明示的な閉じるボタン（`isPresented = false`）を用意する。
- **`path` の型と destination の型がズレる**：`path: [Item]` なら `Item` の destination が必要。複数型を積むなら `NavigationPath`（型消去）を使う。

## 関連
[state.md](./state.md)
