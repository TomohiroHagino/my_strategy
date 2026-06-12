# Flutter 3 実務リファレンス（索引）

> **対象 = Flutter 3.x（Dart 3、null safety前提）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> Flutter の核は「**すべてが Widget**：Widget を組み合わせて画面を作り、状態が変われば再ビルドされる」。

## 核となる考え方
```
 UI = Widget ツリー。状態(State)が変わる → build() が呼ばれ → 画面が再構築される
 （ピクセルは Flutter が自前で描く＝OS部品に依存しない）
```

## 項目（各ファイルへ）

### はじめに / UIの基本
- [getting_started.md](./getting_started.md) … 始め方（flutter create / run / ホットリロード）
- [widgets.md](./widgets.md) … Widget（Stateless / Stateful）とは
- [layout.md](./layout.md) … レイアウト（Row / Column / Container / 制約）とは
- [styling_theming.md](./styling_theming.md) … スタイル / テーマ（Material / Cupertino）とは
- [lists.md](./lists.md) … リスト（ListView.builder）とは

### 状態・画面遷移・ロジック
- [request_flow.md](./request_flow.md) … データの流れ（操作→State→build→描画＋データ取得）とは
- [state_management.md](./state_management.md) … 状態管理（setState / Provider / Riverpod）とは
- [lifecycle.md](./lifecycle.md) … StatefulWidget のライフサイクル（initState / dispose）とは
- [navigation.md](./navigation.md) … ナビゲーション（Navigator / go_router）とは
- [forms_input.md](./forms_input.md) … フォーム / 入力（TextField / Form）とは

### データ・運用
- [async_futures.md](./async_futures.md) … 非同期（Future / FutureBuilder / Stream）とは
- [networking.md](./networking.md) … 通信（http / dio / JSON）とは
- [packages_pubspec.md](./packages_pubspec.md) … パッケージ（pub / pubspec.yaml）とは
- [testing.md](./testing.md) … テスト（widget / unit / integration）とは
- [android_studio.md](./android_studio.md) … Android Studio（Flutter用）とは ← VS Code と並ぶIDE選択肢
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Flutter 3）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
