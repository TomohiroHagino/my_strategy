# React Native 実務リファレンス（索引）

> **対象 = React Native 0.7x（New Architecture / Expo 前提、TypeScript）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> **React の基礎（components / hooks / props / state）は前提**。ここはネイティブ向けの差分（コアコンポーネント・スタイル・ナビゲーション・ネイティブ連携）が中心。
> React基礎 → [../../../フロントエンド/react/react19/](../../../フロントエンド/react/react19/)

## Web React との一番の違い
```
 Web:   <div> <span> <p>   ＋ CSS
 RN :   <View> <Text> <Image> ＋ StyleSheet(flexbox、CSSではない)
        ※ テキストは必ず <Text> の中に。<div>感覚で文字を置くと落ちる。
```

## 項目（各ファイルへ）

### はじめに / UIの基本
- [getting_started.md](./getting_started.md) … 始め方（Expo / CLI / 実行）
- [core_components.md](./core_components.md) … コアコンポーネント（View / Text / Image / ScrollView）とは
- [styling.md](./styling.md) … スタイル（StyleSheet / flexbox）とは
- [components_state.md](./components_state.md) … コンポーネントと状態（Reactの差分）とは
- [lists.md](./lists.md) … リスト（FlatList / SectionList）とは

### 画面遷移・入力・通信
- [request_flow.md](./request_flow.md) … データの流れ（操作→state→再レンダリング→ネイティブUI＋データ取得）とは
- [navigation.md](./navigation.md) … ナビゲーション（React Navigation）とは
- [forms_input.md](./forms_input.md) … 入力（TextInput）とは
- [networking.md](./networking.md) … 通信（fetch / axios）とは

### ネイティブ・状態・運用
- [platform.md](./platform.md) … プラットフォーム差分 / 権限とは
- [native_modules.md](./native_modules.md) … ネイティブ連携（New Architecture / Expo modules）とは
- [storage_state.md](./storage_state.md) … 保存 / 状態管理（AsyncStorage / Redux / Zustand）とは
- [testing.md](./testing.md) … テスト（Jest / RN Testing Library）とは
- [android_studio.md](./android_studio.md) … Android Studio（React Native用）とは ← Android側のSDK/エミュレータ
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（React Native）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
