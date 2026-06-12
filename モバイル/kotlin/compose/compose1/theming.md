# テーマ（Material 3 / MaterialTheme）（Jetpack Compose）

## ひとことで言うと
アプリ全体の **色・文字・形** を一括で定義し、各 `@Composable` から `MaterialTheme.colorScheme` などで参照する仕組み。Material 3（M3）の `MaterialTheme` がその入口で、`colorScheme` / `typography` / `shapes` の3点セットを子ツリー全体に配る。

## 役割・なぜ必要か
- 色やフォントを画面ごとにハードコードすると、ダークテーマ対応やブランド変更時に全箇所を直す羽目になる。`MaterialTheme` に集約しておけば**一箇所変えれば全画面に反映**される。
- M3 は **dynamic color（Android 12+）** をサポートし、ユーザーの壁紙から自動でカラーパレットを生成できる。テーマ層に通しておけば端末の好みに馴染むUIになる。
- ダーク/ライトの切り替え、コントラスト確保、コンポーネント（`Button` / `Card` 等）の既定色も、すべてこのテーマ値を参照して描画される。

## 基本の書き方（コード）
```kotlin
// Color.kt … 色トークン（M3はseed色から生成するのが定番）
val md_primary = Color(0xFF6750A4)
val md_primaryDark = Color(0xFFD0BCFF)

// 1. colorScheme をライト/ダークで用意
private val LightColors = lightColorScheme(primary = md_primary)
private val DarkColors  = darkColorScheme(primary = md_primaryDark)

// 2. typography / shapes
private val AppTypography = Typography(
    titleLarge = TextStyle(fontSize = 22.sp, fontWeight = FontWeight.Bold),
)
private val AppShapes = Shapes(
    medium = RoundedCornerShape(12.dp),
)

// 3. アプリ独自テーマを定義（dynamic color対応込み）
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,            // Android 12+ で壁紙連動
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val ctx = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(ctx) else dynamicLightColorScheme(ctx)
        }
        darkTheme -> DarkColors
        else      -> LightColors
    }
    MaterialTheme(
        colorScheme = colorScheme,
        typography  = AppTypography,
        shapes      = AppShapes,
        content     = content,
    )
}
```

## 実務での使い方・定番パターン
- **エントリで一度だけ包む**：`setContent { AppTheme { /* 画面 */ } }`。以降の全 `@Composable` がテーマ値を継承する。
- **参照は必ずテーマ経由**：色は `MaterialTheme.colorScheme.primary`、文字は `MaterialTheme.typography.titleLarge`、形は `MaterialTheme.shapes.medium`。
```kotlin
@Composable
fun Title(text: String) {
    Text(
        text = text,
        color = MaterialTheme.colorScheme.primary,        // ハードコードしない
        style = MaterialTheme.typography.titleLarge,
    )
}

// 面と前景の対応：背景に primaryContainer を使うなら文字は onPrimaryContainer
Surface(color = MaterialTheme.colorScheme.primaryContainer) {
    Text("ラベル", color = MaterialTheme.colorScheme.onPrimaryContainer)
}
```
- **`on○○` 色の対応を守る**：`primary` の上の文字は `onPrimary`、`surface` の上は `onSurface`。この対で書けばダーク時も自動でコントラストが取れる。
- **ダークテーマは `isSystemInDarkTheme()`** を既定に。手動トグルを足す場合は状態を上げ（ホイスティング）て `darkTheme` 引数へ渡す。→ [state.md](./state.md)
- **dynamic color は端末依存**：壁紙連動を切りたい・ブランド色を固定したいなら `dynamicColor = false` を渡してブランドの `colorScheme` を使う。
- **プレビューでも包む**：`@Preview` 関数内も `AppTheme { ... }` で囲まないと既定テーマで描画され、実機と見た目がズレる。

## ハマりどころ / アンチパターン
- **色やサイズのハードコード**：`Color(0xFF...)` や `16.sp` を各所に直書きすると、ダーク対応・ブランド変更で全滅。必ず `MaterialTheme.colorScheme` / `typography` / `shapes` から引く。
- **M2 と M3 の取り違え**：M2 は `androidx.compose.material`（`colors` / `MaterialTheme.colors.primary`）、M3 は `androidx.compose.material3`（`colorScheme` / `MaterialTheme.colorScheme.primary`）。import を混在させると色トークン名（`onPrimaryContainer` 等は M3 のみ）が食い違う。新規は M3 に統一する。
- **dynamic color を無条件で有効化**：`dynamicLightColorScheme` は **Android 12（API 31）未満で使えない**。`Build.VERSION.SDK_INT >= S` のガード必須。ブランド色を厳密に出したい画面では dynamic を切る判断も要る。
- **`MaterialTheme` で包み忘れ**：包まないと既定テーマで描画され、`MaterialTheme.colorScheme.primary` がブランド色にならない。`setContent` 直下と各プレビューを確認。
- **`on○○` を無視して文字色を固定**：背景だけテーマ追従・文字色は黒固定、にするとダーク時に黒背景＋黒文字で読めなくなる。前景色もテーマの `on○○` から引く。

## 関連: [layout.md](./layout.md) / [state.md](./state.md) / [modifiers.md](./modifiers.md) / [pitfalls.md](./pitfalls.md)
