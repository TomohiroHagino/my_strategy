# 実務でハマる罠まとめ（Pitfalls）（React Native 0.7x）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- RNは「Web Reactの感覚」でそのまま書くと落ちる箇所が多い。Web/ネイティブの差分を症状から該当箇所へ素早く飛ぶための索引。

## コアコンポーネント / UI
- **文字は必ず `<Text>` の中**：`<View>hello</View>` のように文字を直書きすると `Text strings must be rendered within a <Text> component` で落ちる。→ [core_components.md](./core_components.md)
- **HTML要素は無い**：`<div>` / `<span>` / `<p>` / `<button>` は使えない。`<View>` / `<Text>` / `<Pressable>` などRNのコアに置き換える。→ [core_components.md](./core_components.md)
- **`<Image>` はサイズ必須になりがち**：リモート画像は `width`/`height` を指定しないと表示されない（0サイズ）。→ [core_components.md](./core_components.md)

## スタイル / レイアウト
- **`StyleSheet` はCSSではない**：プロパティはcamelCase（`backgroundColor`）、値は数値（`px`なし）、単位やセレクタ・継承の概念は無い。→ [styling.md](./styling.md)
- **`flexDirection` の既定が `column`**：Web（`row`）と逆。横並びにしたいなら明示的に `row` を指定。→ [styling.md](./styling.md)
- **`%` や `vw/vh` は限定的**：レスポンシブは `flex` / `Dimensions` / `useWindowDimensions` で組む。→ [styling.md](./styling.md)
- **`shadow*` はiOSのみ / Androidは `elevation`**：影は両OS分を書かないと片方で出ない。→ [styling.md](./styling.md)

## リスト
- **大量リストは `FlatList`（`ScrollView` でなく）**：`ScrollView` は全要素を一度に描画しメモリを食い落ちる。`FlatList`/`SectionList` で仮想化する。→ [lists.md](./lists.md)
- **`keyExtractor` 必須**：keyが無い/重複すると再描画バグや警告。安定した一意IDを返す。→ [lists.md](./lists.md)
- **`renderItem` 内で重い処理/インライン関数**：再生成でカクつく。`React.memo` / `useCallback` で抑える。→ [lists.md](./lists.md)

## 状態 / コンポーネント（Web React との差分）
- **ブラウザAPIが無い**：`window` / `document` / `localStorage` / `alert` は未定義。RNは `Alert`・`AsyncStorage`・`Dimensions` などに置き換える。→ [components_state.md](./components_state.md)
- **DOM/CSSアニメ前提のライブラリは動かない**：`react-dom` 依存や CSS依存のWeb専用ライブラリはNG。RN対応版を選ぶ。→ [components_state.md](./components_state.md)
- **`Animated` / `Reanimated` の使い分け**：複雑なジェスチャ連動は `Reanimated`。標準 `Animated` で無理に組むとカクつく。→ [components_state.md](./components_state.md)

## 入力 / フォーム
- **`onChange` でなく `onChangeText`**：`TextInput` は `onChangeText={(text) => ...}` で文字列を直接受け取る。`onChange` だとイベントオブジェクトが来て事故る。→ [forms_input.md](./forms_input.md)
- **キーボードが入力欄を隠す**：`KeyboardAvoidingView`（iOSは `behavior="padding"`）や `keyboardShouldPersistTaps` で対処。→ [forms_input.md](./forms_input.md)
- **数値入力も値は文字列**：`keyboardType="numeric"` でも `value` はstring。数値変換は自前で。→ [forms_input.md](./forms_input.md)

## 通信
- **実機から `localhost` は届かない**：`localhost`/`127.0.0.1` は端末自身を指す。実機はマシンのLAN IP、Androidエミュは `10.0.2.2`、`adb reverse tcp:PORT tcp:PORT` も活用。→ [networking.md](./networking.md)
- **iOS ATS / Android cleartext で平文HTTP拒否**：開発で `http://` を使うとブロックされる。原則 `https`、必要なら設定で明示的に許可。→ [networking.md](./networking.md)
- **`fetch` はHTTPエラーで例外を投げない**：4xx/5xxでも `catch` に入らない。`res.ok` を必ず確認。→ [networking.md](./networking.md)

## 保存 / 状態管理
- **`AsyncStorage` は非同期・文字列のみ**：`await` 必須、オブジェクトは `JSON.stringify`/`parse` で出し入れ。同期で読めると思うと事故る。→ [storage_state.md](./storage_state.md)
- **機密情報を `AsyncStorage` に置かない**：トークン・パスワード等は暗号化されない。`expo-secure-store` / Keychain・Keystore を使う。→ [storage_state.md](./storage_state.md)
- **大容量データを `AsyncStorage` に**：サイズ上限・性能の問題。大きいデータはファイル/DB（SQLite等）へ。→ [storage_state.md](./storage_state.md)

## プラットフォーム / 権限
- **SafeArea未対応でノッチ/ホームインジケータに被る**：`SafeAreaView` / `react-native-safe-area-context` で安全領域を確保。→ [platform.md](./platform.md)
- **権限は実行時リクエスト必須**：カメラ・位置情報・通知などは事前に許可を求める。`Info.plist`/`AndroidManifest` の宣言と拒否時のフォールバックも要る。→ [platform.md](./platform.md)
- **`Platform.OS` 分岐の漏れ**：iOS/Androidで挙動が違うAPIは両方検証。片方だけ動くは実機で露見する。→ [platform.md](./platform.md)

## ネイティブ連携 / New Architecture
- **ネイティブ依存はExpo Goで動かない**：カスタムネイティブモジュールを含むライブラリはExpo Goでクラッシュ/未定義。**dev build（development build）**を作る。→ [native_modules.md](./native_modules.md)
- **New Architecture（JSI / Fabric / TurboModules）**：0.7x系で標準化が進行。Bridge前提の古いライブラリは互換性に注意し、対応版/インターロップ層を確認。→ [native_modules.md](./native_modules.md)
- **ネイティブ追加後はリビルド必須**：JSのリロードだけでは反映されない。`pod install` / 再ビルドが要る。→ [native_modules.md](./native_modules.md)

## ビルド / 実行（よくある詰まり）
- **Metroのキャッシュ汚染**：謎の解決エラーは `--reset-cache` / `watchman watch-del-all` で多くが直る。→ [getting_started.md](./getting_started.md)
- **iOS Pods不整合**：依存追加後に `cd ios && pod install`。これ忘れでリンクエラー。→ [getting_started.md](./getting_started.md)

## テスト
- **`act()` 警告・非同期更新**：状態更新は `await` / `findBy*` で待つ。同期前提で書くと flaky に。→ [testing.md](./testing.md)
- **ネイティブモジュールのモック漏れ**：未モックのネイティブ依存でテストが落ちる。`jest.mock` で差し替える。→ [testing.md](./testing.md)

## 関連
[core_components.md](./core_components.md) / [styling.md](./styling.md) / [lists.md](./lists.md) / [forms_input.md](./forms_input.md) / [networking.md](./networking.md) / [components_state.md](./components_state.md) / [storage_state.md](./storage_state.md) / [platform.md](./platform.md) / [native_modules.md](./native_modules.md)
