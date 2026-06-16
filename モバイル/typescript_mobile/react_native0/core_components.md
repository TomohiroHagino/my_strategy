# コアコンポーネント（React Native）

## ひとことで言うと
RNが標準で用意する**画面の部品**。Web の `<div>` `<span>` `<p>` `<img>` は使えず、代わりに **`<View>` / `<Text>` / `<Image>` / `<ScrollView>`** とタップ用の **`<Pressable>`** などを使う。

## 役割・なぜ必要か
- RNはブラウザではなく**ネイティブのUI部品**へ変換される。だから HTMLタグは存在せず、RNが用意したコンポーネントを使う必要がある。Reactの考え方（JSX・props・state）はそのまま、**「タグの語彙」だけがネイティブ用に差し替わる**と理解するとよい。
- 対応の目安:
  - `<div>`（箱・レイアウト） → **`<View>`**
  - `<span>` / `<p>`（文字） → **`<Text>`**
  - `<img>` → **`<Image source={...} />`**
  - スクロール領域 → **`<ScrollView>`**（少量）/ `FlatList`（大量・→ lists.md）
  - クリック/タップ → **`<Pressable>`** または `<TouchableOpacity>`

## 基本の書き方（コード）
```tsx
import { View, Text, Image, ScrollView, Pressable, StyleSheet } from 'react-native';

export function Card() {
  return (
    <View style={styles.card}>
      {/* 文字は必ず <Text> の中に置く */}
      <Text style={styles.title}>こんにちは</Text>

      {/* リモート画像は source={{ uri }} で。width/height が要る */}
      <Image
        source={{ uri: 'https://example.com/a.png' }}
        style={{ width: 80, height: 80 }}
      />

      {/* ローカル画像は require。サイズはファイルから推定される */}
      <Image source={require('./assets/logo.png')} />

      {/* タップ。押下状態に応じてスタイルを返せる */}
      <Pressable
        onPress={() => console.log('tapped')}
        style={({ pressed }) => [styles.btn, pressed && { opacity: 0.6 }]}
      >
        <Text style={styles.btnText}>押す</Text>
      </Pressable>
    </View>
  );
}

// 縦に長い内容はスクロールさせる（件数が多いなら FlatList を使う）
export function Screen() {
  return (
    <ScrollView contentContainerStyle={{ padding: 16 }}>
      <Card />
      <Card />
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  card: { padding: 12, gap: 8 },
  title: { fontSize: 18, fontWeight: 'bold' },
  btn: { backgroundColor: '#3478f6', padding: 10, borderRadius: 8 },
  btnText: { color: 'white', textAlign: 'center' },
});
```

## 実務での使い方・定番パターン
- **`<View>` はレイアウトの箱**。`flex` でほぼすべての配置を組む（→ styling.md）。意味づけは不要、純粋なコンテナ。
- **テキストは全部 `<Text>`**。`<Text>` は入れ子にでき、内側だけ太字・色変えができる（インラインスタイルの継承は `<Text>` の中だけ）。
- **画像の `source`**: リモートは `{{ uri }}`、ローカルは `require('./x.png')`。**リモート画像は `width`/`height` 指定がほぼ必須**（無いと0で表示されない）。
- **タップは `<Pressable>` を第一候補**に。`pressed` で押下中の見た目を作れる。`<TouchableOpacity>` は押すと薄くなる手軽な選択肢。
- **`<SafeAreaView>`**（または `react-native-safe-area-context`）でノッチ/ステータスバーを避ける。

## ハマりどころ / アンチパターン
- **文字を `<Text>` の外に直書きして即クラッシュ**。`<View>こんにちは</View>` は不可。"Text strings must be rendered within a `<Text>` component" が定番エラー。**最頻出のミス**。
- **`<div>` 感覚で `<View>` に文字や `onClick` を期待する**。`<View>` はクリックを受けない（`Pressable` で包む）。イベント名も `onClick` ではなく `onPress`。
- **リモート `<Image>` にサイズを付け忘れて「画像が出ない」**。デフォルトサイズは無い。
- **`<ScrollView>` に大量要素を入れて重くなる**。全要素を一度に描画するため、長いリストは `FlatList`（仮想化）へ。→ lists.md
- **Web由来のタグ（`<button>` `<input>`）を書く**。存在しない。`<Pressable>` / `<TextInput>` を使う。

## 関連
[styling.md](./styling.md) / [lists.md](./lists.md)
