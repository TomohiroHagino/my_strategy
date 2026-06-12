# 実務でハマる罠まとめ（Pitfalls）（Jetpack Compose）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Compose は「宣言的UI＋再コンポーズ」という従来の View と違うモデルで、慣れないと**状態が消える・副作用が多重発火する**といった事故が起きやすい。症状から該当箇所へ素早く飛ぶための索引。

## 状態（State）
- **`remember` しないと状態が消える**：`mutableStateOf(...)` を直書きすると再コンポーズのたび初期値に戻る。`remember { mutableStateOf(...) }` で保持する。→ [state.md](./state.md)
- **構成変更（画面回転）で状態が消える**：`remember` は回転で破棄される。残したい入力値は `rememberSaveable` を使う。→ [state.md](./state.md)
- **状態ホイスティングをしない**：子が自前で状態を持つと親から制御・テストできない。状態を親へ上げ（`value` / `onValueChange`）、子はステートレスにする。→ [state.md](./state.md)
- **画面をまたいで残す/重い状態は ViewModel**：回転や再生成でも保持したい・ロジックを含む状態は `rememberSaveable` でなく ViewModel（`StateFlow`）へ。→ [viewmodel_state.md](./viewmodel_state.md)

## 再コンポーズ / 副作用
- **再コンポーズは複数回・順不同で走る**：`@Composable` 本体で通信・DB書き込み・ログ送信など副作用を直書きすると多重実行される。副作用は Effect API（`LaunchedEffect` / `SideEffect` / `DisposableEffect`）へ。→ [composables.md](./composables.md) / [async_coroutines.md](./async_coroutines.md)
- **`LaunchedEffect` の key 指定ミス**：`key` が変わると再起動、変わらないと再実行されない。一度だけなら `LaunchedEffect(Unit)`、値に追従するなら対象を key に渡す。→ [async_coroutines.md](./async_coroutines.md)

## レイアウト / リスト / Modifier
- **大量要素を `Column` で全部描画**：スクロールしても全件コンポーズされ重い。多数・可変件数は `LazyColumn` / `LazyRow`（`items`）で可視分だけ描画。→ [lists.md](./lists.md)
- **`Modifier` の順序で結果が変わる**：`padding().background()` と `background().padding()` は別物。`clickable` / `clip` / `border` も並び順で挙動が変わる。意図した順に並べる。→ [modifiers.md](./modifiers.md)

## テーマ
- **色やサイズをハードコード**：`Color(0xFF...)` や固定 `sp` 直書きはダーク対応・ブランド変更で全滅。`MaterialTheme.colorScheme` / `typography` / `shapes` から参照する。→ [theming.md](./theming.md)
- **M2 と M3 / dynamic color の取り違え**：M3 は `colorScheme`（`material3`）。dynamic color は Android 12+ のみで SDK ガードが要る。→ [theming.md](./theming.md)

## 関連
[state.md](./state.md) / [viewmodel_state.md](./viewmodel_state.md) / [composables.md](./composables.md) / [async_coroutines.md](./async_coroutines.md) / [lists.md](./lists.md) / [modifiers.md](./modifiers.md) / [theming.md](./theming.md)
