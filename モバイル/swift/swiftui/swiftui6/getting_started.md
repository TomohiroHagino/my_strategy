# 始め方（SwiftUI）

## ひとことで言うと
Xcode で「App」テンプレートから新規プロジェクトを作り、**`App` → `Scene` → `View`** という入れ子で画面を組み立てる、SwiftUI アプリの最小の出発点。書いたコードは `#Preview` で即座に確認できる。

## 役割・なぜ必要か
- SwiftUI は「**画面 = 状態の関数**」という考え方で UI を作る仕組み。まずアプリの**起動点（エントリポイント）**と、最初に表示する**View**を用意する必要がある。
- アプリ起動の流れは `App`（アプリ全体）→ `Scene`（ウィンドウ単位）→ `View`（画面の中身）と階層になっている。この3層を理解すると「どこに何を書くか」が決まる。
- `#Preview` により、シミュレータを起動しなくても **Xcode 上でリアルタイムに見た目を確認**できる。試行錯誤の速度が段違いになるので、SwiftUI を始める最初の一歩として欠かせない。

## 基本の書き方（コード）
```swift
import SwiftUI

// アプリのエントリポイント。@main は「ここから起動」の目印。
@main
struct MyApp: App {
    var body: some Scene {
        // WindowGroup = 標準的なウィンドウ（iPhone なら全画面）
        WindowGroup {
            ContentView()   // 最初に表示する View
        }
    }
}

// 画面の中身。View プロトコルに準拠し、body に見た目を書く。
struct ContentView: View {
    var body: some View {
        VStack(spacing: 12) {
            Image(systemName: "swift")
                .font(.largeTitle)
            Text("Hello, SwiftUI!")
        }
        .padding()
    }
}

// Xcode のキャンバスでライブプレビューされる（実機/シミュレータ不要）
#Preview {
    ContentView()
}
```

## 実務での使い方・定番パターン
- **新規プロジェクト作成**：Xcode → File → New → Project → iOS の「App」を選択。Interface に **SwiftUI**、Language に **Swift** を選ぶ。これで `MyApp.swift`（`@main`）と `ContentView.swift` が自動生成される。
- **起動点はアプリに1つ**：`@main` が付いた `App` 準拠の型は、アプリ全体で**ただ1つ**。ここでアプリ全体の初期設定（後述の `@Environment` 注入や `.modelContainer` など）も行う。
- **`ContentView` は雛形**：自動生成される `ContentView` はただの起点。実際の画面は機能ごとに `HomeView` / `SettingsView` のようにファイルを分けて作るのが定番。→ [views.md](./views.md)
- **`#Preview` を各 View に置く**：開発中の確認は基本これ。引数違いやダークモードなど、複数の `#Preview` を並べて状態ごとに確認できる。
- **シミュレータ / 実機**：実際の動作確認は、Xcode 上部のスキーム選択でデバイス（例：iPhone 15）を選び ▶︎ で実行。`#Preview` は静的確認、シミュレータは挙動確認、と役割を分ける。

## ハマりどころ / アンチパターン
- **Mac ＋ Xcode が必須**：SwiftUI 開発は基本的に **macOS 上の Xcode** でしか行えない（Windows/Linux 単体では不可）。これは前提として最初に押さえる。
- **`#Preview` が動かない**：プレビューはコードを実際にビルド・実行している。**コンパイルエラーがあると即停止**する。プレビューが固まったら、まずビルドエラーを疑う。キャンバスの「Resume」やプレビュー再起動（Option+Cmd+P）で復帰することも多い。
- **`#Preview` に重い処理を書かない**：ネットワーク通信や本物のDBアクセスを伴う View をそのままプレビューすると、止まったり遅くなる。プレビュー用にダミーデータを渡すのが定番。
- **`@main` を2つ書く**：エントリポイントは1つだけ。複数あるとビルドエラー。
- **シミュレータ＝実機ではない**：通知・カメラ・課金など一部はシミュレータで再現できない。最終確認は実機で。

## フォルダ構成（始動直後）
```
MyApp/
├── MyApp/
│   ├── MyAppApp.swift          # @main：アプリの起点
│   ├── ContentView.swift       # 最初に表示する View
│   ├── Assets.xcassets/        # 画像・色のカタログ
│   │   ├── AppIcon.appiconset  #   アプリアイコン
│   │   └── AccentColor.colorset #  アクセントカラー
│   └── Preview Content/        # プレビュー専用リソース
│       └── Preview Assets.xcassets
├── MyApp.xcodeproj/            # プロジェクト設定
│   └── project.pbxproj         #   ビルド構成の本体
├── MyAppTests/                 # ユニットテスト
│   └── MyAppTests.swift
└── MyAppUITests/               # UIテスト
```
- 画面は `View` 構造体（`ContentView` から）。起点は `@main` の `App`。

## 関連
[views.md](./views.md) / [state.md](./state.md)
