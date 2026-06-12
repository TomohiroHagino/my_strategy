# データフロー（@Environment / @Observable）（SwiftUI）

## ひとことで言うと
画面ツリーの**離れた子 View へ、引数バケツリレーなしに値や状態を届ける**仕組み。iOS 17 以降は `@Observable` クラスを `.environment()` で注入し、子で `@Environment(型.self)` として取り出すのが基本形。

## 役割・なぜ必要か
- ログインユーザー・テーマ・カート・設定など「**アプリ全体で共有したい状態や依存**」は、親から子へプロパティを延々と渡す（prop drilling）と煩雑。Environment はその配管を肩代わりする。
- iOS 17 の `@Observable` マクロは、クラスの**変更されたプロパティだけ**を購読 View に通知する。`ObservableObject` + `@Published` 時代より記述が減り、無駄な再描画も減る。
- 「依存を外から注入する（DI）」形になるので、プレビューやテストで**差し替え**やすい。

## 基本の書き方（コード）
```swift
import Observation

@Observable          // iOS 17+：これだけで監視対象クラスになる
final class UserSession {
    var name: String = "ゲスト"
    var isLoggedIn: Bool = false
}

@main
struct MyApp: App {
    @State private var session = UserSession()   // 生存させる所有者は @State

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(session)            // ツリー全体へ注入
        }
    }
}

struct ProfileView: View {
    @Environment(UserSession.self) private var session  // 型で取り出す

    var body: some View {
        // 双方向バインドが要るときは @Bindable で橋渡し
        @Bindable var session = session
        VStack {
            Text("こんにちは \(session.name)")
            TextField("名前", text: $session.name)   // $ は @Bindable 経由
        }
    }
}
```

## 実務での使い方・定番パターン
- **共有状態は `@Observable` クラス + `.environment()`** が定番。所有者（`@State`）はアプリや親 View に1つ置き、子は `@Environment` で参照する。
- **システム値の `@Environment`** も同じ仕組み。`@Environment(\.colorScheme)`、`@Environment(\.dismiss)`、`@Environment(\.locale)` などキーパスで標準値を取れる。
- **`@Bindable`** は、`@Observable` クラスのプロパティへ `$session.name` のように**双方向バインディング**を作りたいときの橋渡し。`TextField` や `Toggle` に渡す。
- **画面ローカルの軽い状態**は Environment に載せず `@State` で十分。共有が要るものだけ Environment へ。→ [state.md](./state.md)
- プレビューでは `.environment(UserSession())` でダミーを注入すると単体表示できる。

## ハマりどころ / アンチパターン
- **注入忘れクラッシュ**：`.environment(session)` を入れていないツリーで `@Environment(UserSession.self)` を読むと**実行時クラッシュ**する（オプショナルでない取得は値が必須）。注入箇所と取得箇所をセットで確認する。
- **iOS 17 が前提**：`@Observable` / `@Bindable` / 値版 `.environment(_:)` は iOS 17 以降。iOS 16 以下では従来の `ObservableObject` + `@EnvironmentObject` を使う。Deployment Target を確認。
- **`@State` を付け忘れる**：`@Observable` クラスを子へ渡したいだけなら所有は不要だが、**生成して保持する側**は `@State private var x = Model()` にしないとライフサイクルが安定しない。
- **`@Bindable` を使わず `$` を書く**とコンパイルエラー。`@Observable` クラスのプロパティに `$` で束ねるには `@Bindable var model = model` のひと手間が要る。
- **何でも Environment に載せる**設計はテスト・追跡を難しくする。共有が本当に必要な依存に絞る。→ [coding-style] 高凝集・低結合。

## 関連
[state.md](./state.md) / [navigation.md](./navigation.md) / [async_data.md](./async_data.md)
