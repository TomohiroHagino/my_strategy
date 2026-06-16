# React Native

## 一言で
**React で iOS / Android アプリを書く**フレームワーク。WebView ではなく**本物のネイティブUIコンポーネント**を動かす。言語は JS/TS＋React なので、**React の知識（コンポーネント・hooks・JSX）をそのまま流用**できる。入口は **Expo** が主流。

## 特徴
- **Reactそのまま**：関数コンポーネント・hooks・props/state・JSX。
- **ネイティブUIにマッピング**：`<View>`→ネイティブのコンテナ、`<Text>`→ネイティブのテキスト。
- **StyleSheet（flexbox）**：CSSではないがflexboxベースのスタイル。
- **React Navigation** で画面遷移、**Expo** で開発・ビルド・配布を簡単に。
- **New Architecture**（JSI / Fabric / TurboModules）が新しい標準。

## Flutter との違い
| | React Native | Flutter |
|---|---|---|
| 言語 | JS/TS（React） | Dart |
| UI | **ネイティブUI**をブリッジ | 自前描画 |
| 既存資産 | **React/JS知識を流用** | Dart新規 |
| 見た目 | OSネイティブ寄り | 全OSで一致 |

## このフォルダの構成
- [react_native0/](./react_native0/) … **React Native 実務リファレンス（フラッグシップ）**。始め方〜コアコンポーネント〜スタイル〜ナビゲーション〜ネイティブ連携〜テスト〜罠まで、項目=1ファイル。
  - ※ RN は 0.x 系。フォルダ名 `react_native0` はメジャー番号0（rails7 等と同じ命名規則）。
- [落とし穴.md](./落とし穴.md) … **JS/TSの仕組みがhooks/モバイル文脈でどう牙を剥くか**（stale closure・非同期setState・依存配列ミス・アンマウント後setState・ブリッジのシリアライズ制約・浮動小数/型・FlatList性能）。JS共通は [../../フロントエンド/javascript/落とし穴.md](../../フロントエンド/javascript/落とし穴.md)、RN/React固有は [react_native0/pitfalls.md](./react_native0/pitfalls.md)。

> 前提: **React の基礎**（components / hooks / state）は [../../フロントエンド/react/react19/](../../フロントエンド/react/react19/) を参照。RNはそこにネイティブ向けの差分を足したもの。
