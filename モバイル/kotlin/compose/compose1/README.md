# Jetpack Compose 実務リファレンス（索引）

> **対象 = Jetpack Compose 1.x（Kotlin / Android Studio / Material 3）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> 核は「**UIは`@Composable`関数**：状態が変われば再コンポーズ（再描画）される」。

## 核となる考え方
```
 remember { mutableStateOf(...) } の状態が変わる → 関連する @Composable が再コンポーズ
 → 画面が状態に追従。手でViewを更新しない。
```

## 項目（各ファイルへ）

### はじめに / UIの基本
- [getting_started.md](./getting_started.md) … 始め方（Android Studio / setContent / プレビュー）
- [composables.md](./composables.md) … `@Composable`（関数・再コンポーズ）とは
- [layout.md](./layout.md) … レイアウト（Column / Row / Box / Modifier）とは
- [modifiers.md](./modifiers.md) … Modifier（チェーン・順序）とは
- [lists.md](./lists.md) … リスト（LazyColumn / items）とは

### 状態・画面遷移・ロジック
- [request_flow.md](./request_flow.md) … データの流れ（操作→State→再コンポーズ→UI＋データ取得）とは
- [state.md](./state.md) … 状態（remember / mutableStateOf / 状態ホイスティング）とは
- [viewmodel_state.md](./viewmodel_state.md) … ViewModel / StateFlow とは
- [navigation.md](./navigation.md) … ナビゲーション（Navigation Compose）とは

### 非同期・テーマ・運用
- [async_coroutines.md](./async_coroutines.md) … 非同期（コルーチン / LaunchedEffect）とは
- [theming.md](./theming.md) … テーマ（Material 3 / MaterialTheme）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Jetpack Compose）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
