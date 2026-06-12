# Flutter

## 一言で
Google の**クロスプラットフォームUIフレームワーク**。**1つの Dart コードから iOS / Android / Web / Desktop** を作る。**すべてが Widget**で、独自描画エンジン（Skia / Impeller）で画面を自前で描く＝ピクセル単位の制御とネイティブ並みの性能。**ホットリロード**で開発が速い。

## 特徴
- **すべてが Widget**：UIも余白もテーマも Widget。Widget ツリーで画面を組む。
- **宣言的UI**：状態から画面を宣言（`StatelessWidget` / `StatefulWidget`）。
- **自前描画**：OSのUI部品を使わず自分で描く（見た目が全プラットフォームで一致）。
- **Material / Cupertino** の既製デザイン。
- **ホットリロード**で即時反映。

## React Native との違い
| | Flutter | React Native |
|---|---|---|
| 言語 | Dart | JS/TS（React） |
| UI | **自前描画**（独自エンジン） | **ネイティブUI**をブリッジ |
| 一貫性 | 全OSで同じ見た目 | OSネイティブの見た目に寄る |
| 既存資産 | Dart新規 | React/JS知識を流用 |

## このフォルダの構成
- [flutter3/](./flutter3/) … **Flutter 3 実務リファレンス（フラッグシップ）**。始め方〜Widget〜レイアウト〜状態管理〜ナビゲーション〜非同期〜テスト〜罠まで、項目=1ファイル。

> 関連: 言語の土台は [../README.md](../README.md)（Dart）。
