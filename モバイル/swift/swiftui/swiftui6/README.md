# SwiftUI 実務リファレンス（索引）

> **対象 = SwiftUI（Swift 6 / iOS 17〜18 era、Xcode）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> 核は「**View は状態の関数**：`@State` 等が変わると、それを使う View だけ再描画される」。

## 核となる考え方
```
 @State などの状態が変わる → body が再評価される → 画面が状態に追従
 View は struct（値型）。手で更新せず「状態を変える」。
```

## 項目（各ファイルへ）

### はじめに / UIの基本
- [ハンズオン.md](./ハンズオン.md) … 【実習】Xcodeで環境構築→シミュレータで画面を出す→@State→List（手を動かす1回目／macOS必須）
- [getting_started.md](./getting_started.md) … 始め方（Xcodeプロジェクト / @main App / プレビュー）
- [views.md](./views.md) … View（`struct` / `body` / 合成）とは
- [layout.md](./layout.md) … レイアウト（VStack / HStack / ZStack / frame）とは
- [modifiers.md](./modifiers.md) … modifier（`.padding()` 等・順序）とは
- [lists.md](./lists.md) … リスト（List / ForEach / Identifiable）とは

### 状態・画面遷移・データ
- [request_flow.md](./request_flow.md) … データの流れ（操作→State→body再評価→View＋データ取得）とは
- [state.md](./state.md) … 状態（@State / @Binding / @Observable）とは
- [data_flow.md](./data_flow.md) … データフロー（@Environment / @Observable）とは
- [navigation.md](./navigation.md) … ナビゲーション（NavigationStack / sheet）とは
- [forms_input.md](./forms_input.md) … 入力（TextField / Form）とは

### データ取得・運用
- [async_data.md](./async_data.md) … 非同期（async/await / .task / URLSession）とは
- [viewinspector.md](./viewinspector.md) … ViewInspector（SwiftUIのView単体テスト：inspect() でView階層を検査）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（SwiftUI）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
