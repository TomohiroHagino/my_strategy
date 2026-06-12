# 入力（TextField / Form）（SwiftUI）

## ひとことで言うと
ユーザーからの**文字・選択・スイッチ等の入力を受け取る部品群**。中心は `TextField`。`@State` で持った値を `$`（ドル記号）で**双方向バインディング**として渡し、入力に追従して状態が変わる。

## 役割・なぜ必要か
- SwiftUI は「View は状態の関数」。入力部品も例外でなく、**入力＝状態の書き換え**として扱う。手で `text` を取りに行くのではなく、状態（`@State`）を共有して自動同期させる。
- `Form` は設定画面のような入力の集まりを、iOS 標準の見た目（区切り・グルーピング）で整える専用コンテナ。
- 多数の入力部品（`Toggle` / `Picker` / `Stepper` / `DatePicker`）が同じ「`$` でバインディング」という統一ルールで動くので、覚えることが少ない。

## 基本の書き方（コード）
```swift
struct ProfileForm: View {
    // 入力値は View が所有する @State（private 推奨）
    @State private var name = ""
    @State private var agreed = false
    @State private var plan = Plan.free
    @State private var count = 1
    @State private var birthday = Date()

    // フォーカス（どの入力欄が選択中か）を管理する
    @FocusState private var nameFocused: Bool

    enum Plan: String, CaseIterable, Identifiable {
        case free, pro
        var id: Self { self }
    }

    var body: some View {
        Form {
            Section("基本情報") {
                // $name で「読み書き両用のバインディング」を渡す
                TextField("名前", text: $name)
                    .focused($nameFocused)
                    .submitLabel(.done)
            }
            Section("設定") {
                Toggle("規約に同意", isOn: $agreed)
                Picker("プラン", selection: $plan) {
                    ForEach(Plan.allCases) { p in
                        Text(p.rawValue).tag(p)
                    }
                }
                Stepper("数量：\(count)", value: $count, in: 1...10)
                DatePicker("誕生日", selection: $birthday,
                           displayedComponents: .date)
            }
        }
        .onAppear { nameFocused = true } // 表示時に名前欄へフォーカス
    }
}
```

## 実務での使い方・定番パターン
- **`$` でバインディングを渡す**：`text:` `isOn:` `selection:` `value:` `selection:` いずれも `$状態名`。`$` を付け忘れると「`Binding<String>` が必要なのに `String` を渡した」型エラーになる。
- **`Form` vs 素の `VStack`**：設定風 UI なら `Form`。`Section("見出し")` でグルーピングでき、iOS らしい区切り線・背景が自動で付く。自由レイアウトなら `VStack` を使い分ける。
- **`Picker` には `.tag()` が要る**：`selection` の型と各行の `tag` の型を一致させる。`enum` を `Identifiable + CaseIterable` にして `ForEach` で回すのが定番。
- **キーボード制御**：`.keyboardType(.emailAddress)` で種類、`.textInputAutocapitalization(.never)` で自動大文字化抑制、`.submitLabel(.done)` で改行キーの文言を変更。
- **`@FocusState` でフォーカス制御**：`Bool` 版（単一）か、列挙体版（複数欄の切替）。`.focused($state)` を各 `TextField` に付け、`state = ...` で能動的に移動・解除できる（キーボードを閉じる＝ `nameFocused = false`）。
- **`@FocusState` 列挙体版**：複数欄を `enum Field { case name, email }` で識別し、Enter で次欄へ送る等の遷移が書ける。
- **数値入力**：`TextField("金額", value: $amount, format: .number)` で `Int`/`Double` に直接バインドできる（`text:` 版は `String` 経由なので変換が要る）。

## ハマりどころ / アンチパターン
- **`$` の付け忘れ／付け過ぎ**：状態を渡すときは `$name`、ただ値を読むだけ（表示）なら `name`。混同が型エラーの最頻原因。→ [state.md](./state.md)
- **`@State` を子 View へ「値」で渡してしまう**：子で編集させたいなら親が `@State`、子は `@Binding` で受け、呼び出し側は `$` で渡す。→ [state.md](./state.md)
- **`Form` の見た目が想定と違う**：`Form` は iOS 標準スタイルを強制する。色・余白を細かく作り込みたい画面に `Form` を使うと戦いになる。自由 UI は `VStack`/`ScrollView` で。
- **`Picker` の選択が反映されない**：`tag` の型と `selection` の型不一致が原因のことが多い。`enum` の `id`/`tag` を揃える。
- **キーボードが入力欄を隠す**：`Form`/`List` 内なら自動スクロールされやすいが、自前レイアウトでは `.scrollDismissesKeyboard(.interactively)` や `ScrollView` 化で回避。
- **`DatePicker` の `displayedComponents` 指定漏れ**：日付だけ欲しいのに時刻も出る。`.date` / `.hourAndMinute` を明示。
- **フォーカスを `@State` で持とうとする**：フォーカスは専用の `@FocusState` を使う。`@State Bool` では `.focused()` に渡せない。

## 関連: [state.md](./state.md) / [data_flow.md](./data_flow.md) / [navigation.md](./navigation.md)
