# データの流れ・各部分は何を返すか（SwiftUI）

## ひとことで言うと
モバイルは「サーバのリクエスト層」ではなく、**状態→UIの一方向データフロー**で考える。**ユーザー操作 → State が変わる → body が再評価される → View が更新される**。データ取得（async/await）は State（`@Published` 等）を更新する手段で、最終的にすべて State 経由で画面に反映される。

## 全体の流れ（図）
```
ユーザー操作（タップ／入力）
   │
   ▼
[State 更新]   @State の値変更 / ObservableObject の @Published を更新
   │           ※ 手で View を更新しない。状態を変えるだけ
   ▼
[body 再評価]  その State を使う View の body が再実行される
   │
   ▼
[View 更新]    返した View 構造を SwiftUI が差分描画
   │
   ▼
  画面（描画）

── データ取得フロー（横入り）──────────────
[API / URLSession]  async/await で取得
   │ データ（モデル）を返す
   ▼
[State 更新]  @Published に格納（.task / Task 内で取得 → 代入、MainActor）
   │
   ▼  以降は上と同じ：body 再評価 → View 更新
```
一方向（状態 → UI）。View から State への戻り矢印は無く、操作は「次の状態」を作るだけ。

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 役割 |
|---|---|---|---|
| **ユーザー操作** | タップ／入力 | アクション（State 変更） | 状態変更のきっかけ |
| **API / URLSession** | URL / 条件 | **データ（モデル）**（async の戻り値） | 外部からデータ取得 |
| **State / @Published** | 操作・取得結果 | **body へ渡る現在値** | 画面の元になる唯一の真実 |
| **body** | State | **View（値型 struct）** | 状態から画面を組み立てる |
| **View** | — | 描画される UI そのもの | 運ばれる側（値型・毎回生成） |

- **View は状態の関数**：`@State` 等が変わると、それを使う View だけ再描画される。
- **View は struct（値型）**：手で更新せず「状態を変える」。差分は SwiftUI が計算する。

## コードで通して見る
```swift
// パターンA：@State ＋ 操作
struct CounterView: View {
  @State private var count = 0                  // State（唯一の真実）

  var body: some View {                          // State を受け取り…
    VStack {
      Text("\(count)")                           // …View（UI）を返す
      Button("+1") { count += 1 }                // ユーザー操作 → State 更新（body 再評価）
    }
  }
}
```

```swift
// パターンB：@Observable ＋ async/await（データ取得フロー）
@Observable
final class UserModel {
  var user: User?
  var loading = false

  @MainActor
  func load() async {
    loading = true
    user = try? await api.fetchUser()            // API → データを返す → @Published 相当を更新
    loading = false
  }
}

struct UserView: View {
  @State private var model = UserModel()

  var body: some View {
    Group {
      if model.loading { ProgressView() }
      else if let user = model.user { Text(user.name) }  // 状態 → View
    }
    .task { await model.load() }                  // 画面表示時に取得（状態を更新）
  }
}
```

## 実務での使い方・定番パターン
- **画面の状態は `@Observable`（または ObservableObject）に集約**：`@State` のローカル値と使い分け、複雑な画面はモデルに寄せる。→ [state.md](./state.md) / [data_flow.md](./data_flow.md)
- **取得は async/await → 状態 → body**：`.task` で表示時に取得し、loading/error も状態として持つ。UI 更新は `@MainActor`。→ [async_data.md](./async_data.md)
- **View を直接いじらない**：常に「状態を変える」→ SwiftUI が body を再評価。命令的更新の発想を持ち込まない。

## ハマりどころ / アンチパターン
- **body の中で副作用**：body で API 呼び出し・状態更新すると再評価ループ。取得は `.task` / `Task` で。
- **メインスレッド外で State 更新**：UI 反映は `@MainActor` 必須。バックグラウンドで `@Published` を触ると警告／不整合。
- **値型 View を参照型のように扱う**：View に可変状態を直に持たせない。`@State`/`@Observable` を通す。
- **`.task` を使わず onAppear で同期取得**：UI を止める。非同期は `.task`（自動キャンセル付き）が定番。

## 関連
[state.md](./state.md) / [data_flow.md](./data_flow.md) / [async_data.md](./async_data.md) / [views.md](./views.md)
