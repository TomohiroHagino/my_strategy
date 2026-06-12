# 非同期データ取得（async/await / .task / URLSession）（SwiftUI）

## ひとことで言うと
ネットワークなど**時間のかかる処理を待っている間も画面を止めない**ための仕組み。`async`/`await` で非同期処理を書き、View の `.task { }` で起動し、`URLSession` で取得して `Codable` で JSON を変換、UI 更新は `@MainActor` 上で行う。

## 役割・なぜ必要か
- API からデータを取る間 UI をブロックすると操作不能になる。`async`/`await` は「待つけど止めない」コードを**同期処理のように直線的**に書ける。
- `.task { }` は **View が表示された時に非同期処理を開始し、View が消えると自動でキャンセル**してくれる。手動キャンセル管理が不要。
- UI（`@State` など）の更新は**メインスレッド**で行う必要があり、Swift 6 ではこれを `@MainActor` でコンパイル時に強制できる（データ競合を未然に防ぐ）。

## 基本の書き方（コード）
```swift
struct Post: Codable, Identifiable {   // JSON → 構造体へ自動変換
    let id: Int
    let title: String
}

@MainActor                              // この型の更新はメインアクター上で
@Observable
final class PostStore {
    var posts: [Post] = []
    var errorMessage: String?

    func load() async {
        do {
            let url = URL(string: "https://example.com/posts")!
            let (data, _) = try await URLSession.shared.data(from: url)  // await で待つ
            posts = try JSONDecoder().decode([Post].self, from: data)    // Codable で復元
        } catch {
            errorMessage = error.localizedDescription   // 失敗は握り潰さず表示
        }
    }
}

struct PostListView: View {
    @State private var store = PostStore()

    var body: some View {
        List(store.posts) { Text($0.title) }
            .task { await store.load() }   // 表示時に開始・離脱で自動キャンセル
    }
}
```

## 実務での使い方・定番パターン
- **取得開始は `.task { }`**。`onAppear` でも呼べるが、`onAppear` は同期クロージャなので `Task { }` を自分で作る必要があり、キャンセルも手動。非同期なら `.task` が素直。
- **`.task(id:)`** を使うと、`id` が変わるたびに前のタスクを自動キャンセルして再実行できる（検索語の変化に追従するなど）。
- **状態は3つ持つ**のが定番：`loading` / `data` / `error`。読み込み中はプログレス、失敗はリトライ表示。
- **`Codable`** で JSON ↔ 構造体を変換。キー名が違うときは `CodingKeys` で対応付け、日付は `decoder.dateDecodingStrategy` を設定。
- **UI 更新は `@MainActor`**。ストアや更新メソッドに `@MainActor` を付け、バックグラウンドで取得しても反映はメインで行う。→ [state.md](./state.md)

## ハマりどころ / アンチパターン
- **`.task` と `onAppear` の混同**：非同期 API を呼ぶなら `.task`。`onAppear` 内で `await` は直接書けず、`Task { }` 包みとキャンセル管理が必要になる。
- **UI 更新をメイン外で行う**と、Swift 6 の並行性チェックで**コンパイルエラー**になる（旧バージョンでは実行時にちらつき・クラッシュ）。`@State` を触る処理は `@MainActor` に載せる。
- **Swift 6 の Sendable 警告**：アクターをまたいで渡す型が `Sendable` でないと警告/エラー。モデルは `struct` + `Codable` にすると素直に `Sendable` になりやすい。
- **エラーの握り潰し**：`try?` で握って空配列のまま、は原因不明の「データが出ない」を生む。`do/catch` で必ず捕捉し、ユーザーに見える形で扱う。→ [coding-style] 「エラーを黙って飲み込まない」。
- **キャンセル無視のループ**：長い処理では `try Task.checkCancellation()` を挟む。`.task` の自動キャンセルも、これを見ないと止まらない。
- **`URL(string:)!` の強制アンラップ**：固定URL以外では失敗し得る。動的に組むなら `guard let` で安全に。

## 関連
[state.md](./state.md) / [data_flow.md](./data_flow.md) / [pitfalls.md](./pitfalls.md)
