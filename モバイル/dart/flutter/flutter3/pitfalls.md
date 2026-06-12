# 実務でハマる罠まとめ（Pitfalls）（Flutter 3）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Flutterは「動く書き方」と「事故る書き方」が紙一重なものが多い。症状から該当箇所へ素早く飛ぶための索引。

## レイアウト / 描画
- **"RenderFlex overflow"（黄黒の縞）**：`Row`/`Column` の子が画面幅・高さを超過。`Expanded`/`Flexible` で割り当てるか、`SingleChildScrollView`/`Wrap` でスクロール・折返しに。→ [layout.md](./layout.md)

## Widget / 再ビルド
- **`setState` 乱用・巨大Widget**：1つの `setState` でツリー全体が再ビルドされ重い。状態を持つ範囲を最小Widgetに切り出し、規模が出たらProvider/Riverpodへ。→ [state_management.md](./state_management.md)
- **`const` 付与で再ビルド抑制**：不変なWidgetは `const` を付けると再生成されずキャッシュされ高速。付け忘れは無駄な再ビルドの定番。→ [widgets.md](./widgets.md)
- **Stateful / Stateless の選択**：内部に変化する状態が無いなら `StatelessWidget`。なんでも `Stateful` にすると無駄に重く複雑。→ [widgets.md](./widgets.md)

## ライフサイクル / リソース
- **controllerの `dispose` 忘れ**：`TextEditingController`/`AnimationController`/`ScrollController` 等を破棄せずメモリリーク。`State.dispose()` で必ず `dispose()`。→ [lifecycle.md](./lifecycle.md) / [forms_input.md](./forms_input.md)
- **async後 `setState` の `mounted` 確認**：`await` 中に画面が破棄されると "setState() called after dispose()" 例外。`if (!mounted) return;` を挟む。→ [lifecycle.md](./lifecycle.md)

## 非同期 / データ取得
- **`FutureBuilder` の `build` 内で `Future` を毎回生成**：再ビルドのたびに新しい `Future` が走り、ローディングが終わらない/通信が多重に。`Future` は `initState` 等で一度だけ作り保持する。→ [async_futures.md](./async_futures.md)

## リスト
- **大量リストは `.builder`**：`ListView(children: [...])` は全要素を即時生成して重い。`ListView.builder` で表示分だけ遅延生成。→ [lists.md](./lists.md)

## スタイル / テーマ
- **`Theme.of(context)`**：色・文字スタイルは直書きせずテーマから取得。`context` が無い場所（初期化前）で呼ぶとエラーになる点に注意。→ [styling_theming.md](./styling_theming.md)

## パッケージ / 設定
- **pubspecのインデント**：`pubspec.yaml` はYAMLでスペース2字・タブ禁止。インデント崩れで依存解決が黙って失敗・無効化する。→ [packages_pubspec.md](./packages_pubspec.md)

## テスト
- **`pumpWidget` 前の `find`/`tap`** と **`pump`/`pumpAndSettle` の取り違え**：描画前操作は失敗、無限アニメに `pumpAndSettle` はタイムアウト。→ [testing.md](./testing.md)

## 関連
[layout.md](./layout.md) / [widgets.md](./widgets.md) / [state_management.md](./state_management.md) / [lifecycle.md](./lifecycle.md) / [async_futures.md](./async_futures.md) / [lists.md](./lists.md) / [testing.md](./testing.md)
