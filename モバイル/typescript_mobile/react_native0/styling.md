# スタイル（React Native）

## ひとことで言うと
RNの見た目を決める仕組み。**`StyleSheet.create` でオブジェクトとして書く**。CSSファイルではなく、**flexboxベース・キャメルケース・単位なし(dp)** という独自ルールを持つ「CSS風JS」。

## 役割・なぜ必要か
- RNにはブラウザのCSSエンジンが無い。代わりに**JSオブジェクトでスタイルを表現**し、`style` propでコンポーネントに渡す。`className` もグローバルCSSも基本的に存在しない。
- レイアウトは **flexbox が中心**。しかも Webと既定が違い、**`flexDirection` の初期値が `column`（縦並び）**。Webの `row`（横並び）と逆なので、ここが最初の落とし穴。
- スタイルは**カスケード（継承・グローバル適用）しない**。各コンポーネントに明示的に渡す。例外的に `<Text>` の中の文字系プロパティだけは入れ子に継承される。

## 基本の書き方（コード）
```tsx
import { View, Text, StyleSheet } from 'react-native';

export function Box() {
  return (
    // 配列でスタイルを合成（後勝ち）。条件付きも書ける
    <View style={[styles.box, styles.shadow, true && { borderWidth: 1 }]}>
      <Text style={styles.label}>合成スタイル</Text>
      {/* インラインも可（再生成コストがあるので多用は避ける） */}
      <Text style={{ marginTop: 4, color: '#666' }}>インライン</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  box: {
    // 既定は flexDirection: 'column'（縦並び）。横並びは明示する
    flexDirection: 'row',
    justifyContent: 'space-between', // 主軸方向の整列
    alignItems: 'center',            // 交差軸方向の整列
    gap: 8,                          // 子要素の間隔
    padding: 16,                     // 単位なし = dp（"16px" ではない）
    backgroundColor: '#fff',
    borderRadius: 12,
  },
  shadow: {
    // iOS と Android で影の指定が分かれる
    shadowColor: '#000', shadowOpacity: 0.1, shadowRadius: 6, // iOS
    elevation: 3,                                             // Android
  },
  label: { fontSize: 16, fontWeight: '600' }, // キャメルケース
});
```

```tsx
// flex:1 は「親の余白を埋める」。画面いっぱいに広げる定番
const layout = StyleSheet.create({
  screen: { flex: 1, backgroundColor: '#f2f2f7' },
});
```

## 実務での使い方・定番パターン
- **`StyleSheet.create` にまとめる**。命名で意図が伝わり、`style={[base, variant]}` で**バリエーションを合成**できる（配列は後ろが優先）。
- **レイアウトは flex で組む**: `flex: 1` で伸縮、`flexDirection` で並び、`justifyContent`/`alignItems` で整列、`gap` で間隔。Webと同じ語彙だが**既定が縦**。
- **プラットフォーム分岐**: `Platform.OS === 'ios'` や `Platform.select({ ios, android })` で差を吸収（影・余白など）。→ platform.md
- **画面サイズ対応**: `useWindowDimensions()` で幅を取り、`%` や計算値で可変に。
- **NativeWind**（Tailwind風 `className`）を入れる選択肢もあるが、まずは素の `StyleSheet` を理解しておく。

## ハマりどころ / アンチパターン
- **CSSの感覚で書いて動かない**: `"16px"` のような単位付き文字列は不可（数値=dp）。`background-color` のようなケバブケースも不可（`backgroundColor`）。`box-shadow` も無い。
- **`flexDirection` 既定が `column`** を忘れ、「横に並べたのに縦に積まれる」。横並びは `flexDirection: 'row'` を明示。**最頻出の混乱**。
- **グローバルCSSを期待する**: セレクタも継承（カスケード）も無い。各所に明示的に `style` を渡す。共通化はオブジェクトの再利用で行う。
- **インラインスタイルを大量に書いて再レンダリングが重い**: 毎回新しいオブジェクトが生成される。固定値は `StyleSheet.create` 側へ。
- **影が片方のOSで出ない**: iOSは `shadow*`、Androidは `elevation`。両方書く。
- **`%` 高さが効かない**: 親に明確な高さ（または `flex`）が無いと `height: '100%'` は計算できない。

## 関連
[core_components.md](./core_components.md)
