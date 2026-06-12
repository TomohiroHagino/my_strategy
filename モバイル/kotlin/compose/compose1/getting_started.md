# 始め方（Jetpack Compose）

## ひとことで言うと
Jetpack Compose を使う最初の一歩。**Android Studio で Compose プロジェクトを作り、`setContent { }` に `@Composable` 関数を渡して画面を組む**までの土台。

## 役割・なぜ必要か
- Compose は「XMLレイアウト＋findViewById」ではなく、**Kotlin の関数で UI を宣言的に書く**仕組み。まずこの書き方の入口を押さえる。
- 画面の起点は `Activity` の `setContent { }`。ここに渡した `@Composable` がそのまま画面になる。手で View を組み立てる必要がない。
- `@Preview` で **アプリを起動せず Android Studio 上で見た目を確認**できる。試行錯誤が速くなるのが Compose の大きな利点。
- Compose は Gradle のプラグイン・依存・コンパイラに密に乗っているため、**プロジェクト初期設定（Gradle）が正しいこと**が前提になる。

## 基本の書き方（コード）
```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            // アプリ全体のテーマでラップ（色・タイポグラフィを供給）
            AppTheme {
                // Material 3 の土台。背景色などを敷く
                Surface {
                    Greeting(name = "Compose")
                }
            }
        }
    }
}

// UI は「関数」。引数を受け取り Text を描くだけ
@Composable
fun Greeting(name: String) {
    Text("Hi $name")
}

// アプリを起動せず IDE で見た目確認
@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    AppTheme {
        Greeting(name = "Compose")
    }
}
```

```kotlin
// app/build.gradle.kts（要点）
android {
    buildFeatures {
        compose = true            // Compose を有効化
    }
}
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2024.09.00")
    implementation(composeBom)    // バージョンを束ねる BOM
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    debugImplementation("androidx.compose.ui:ui-tooling") // @Preview 用
}
```

## 実務での使い方・定番パターン
- **新規プロジェクトは「Empty Activity」テンプレ**から作るのが定番。最初から Compose / Material 3 / テーマ（`AppTheme`）が組み込まれた雛形が出る。
- 画面の最上位は **`AppTheme { Surface { ... } }`** でラップする。色・形・タイポグラフィをアプリ全体で一貫させるため。→ [theming.md](./theming.md)
- 開発中は **`@Preview` を多用**。引数違いのプレビューを並べて、状態ごとの見た目（空・読み込み中・エラー）を一覧確認する。
- BOM（`compose-bom`）で各 Compose ライブラリのバージョンを束ねる。個別にバージョンを書かないのが今の定番。
- まず小さい `@Composable` を1つ作って `@Preview` で表示 → そこから状態やレイアウトを足していく、が安全な進め方。→ [composables.md](./composables.md)

## ハマりどころ / アンチパターン
- **Android Studio が実質必須**。Compose は IDE のプレビュー・Compose コンパイラ・Gradle 連携と一体で、テキストエディタ＋コマンドだけの開発は現実的でない。最新安定版の Android Studio を使う。
- **Gradle 同期忘れ / 失敗**。`build.gradle.kts` を触ったら必ず "Sync Now"。Kotlin・Compose コンパイラ・BOM のバージョン不整合でビルドが落ちるのが定番。エラーは大抵バージョン不一致。
- **`@Preview` が出ない/更新されない**：`ui-tooling` を `debugImplementation` に入れ忘れ、関数に引数が必須、`@Composable` を付け忘れ、などが原因。プレビュー対象の関数は**引数なし or 既定値あり**にする。
- `@Preview` 関数に**重いロジックや本番のデータ取得を書かない**。プレビューはダミーデータで描くだけにする。
- テンプレ生成された `AppTheme` を**剥がして直書きしない**。テーマ経由で色を取る癖を最初からつける。→ [theming.md](./theming.md)

## フォルダ構成（始動直後）
```
MyApp/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/example/myapp/
│   │   │   │   ├── MainActivity.kt   # setContent { } の起点
│   │   │   │   └── ui/theme/         # テーマ定義
│   │   │   │       ├── Color.kt      #   色
│   │   │   │       ├── Theme.kt      #   AppTheme 本体
│   │   │   │       └── Type.kt       #   タイポグラフィ
│   │   │   ├── res/                  # リソース
│   │   │   │   ├── values/           #   文字列・テーマ
│   │   │   │   ├── drawable/         #   ベクター画像
│   │   │   │   └── mipmap/           #   アプリアイコン
│   │   │   └── AndroidManifest.xml   # アプリ宣言
│   │   ├── test/                     # ユニットテスト（JVM）
│   │   └── androidTest/              # 計装テスト（端末）
│   ├── build.gradle.kts             # アプリ単位の依存
│   └── proguard-rules.pro           # リリース時の難読化ルール
├── build.gradle.kts                 # プロジェクト全体
├── settings.gradle.kts              # モジュール構成
├── gradle/
│   └── libs.versions.toml           # バージョンカタログ
├── gradlew                          # Gradle ラッパー
├── gradle.properties                # Gradle 設定
└── local.properties                 # SDK パス（コミットしない）
```
- コードは `app/src/main/java/（パッケージ）`。Composeは `@Composable` 関数で画面を書く。

## 関連
[composables.md](./composables.md) / [state.md](./state.md)
