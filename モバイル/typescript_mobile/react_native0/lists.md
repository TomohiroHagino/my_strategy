# リスト（React Native）

## ひとことで言うと
たくさんの要素を縦（横）に並べて表示する仕組み。RN では用途で使い分ける：**大量データは `FlatList`**（遅延描画＝画面に映る分だけ描く）、見出し付きのグループは `SectionList`、**少数を一気に出すだけなら `ScrollView`**（全要素を一度に描画）。

## 役割・なぜ必要か
- モバイルは縦スクロールのリストが主役（タイムライン・一覧・設定…）。これを素朴に全部描くとメモリと描画コストで死ぬ。
- `FlatList` は **ビューポート（見えている範囲）だけを描画し、スクロールに応じて作り直す**仮想化（virtualization）をやってくれる。Web の react-window / react-virtualized 相当が標準で入っているイメージ。
- 逆に `ScrollView` は**中身を全部マウントする**ので、数十件までの固定的な内容（フォーム・設定画面）向き。

## 基本の書き方（コード）
```tsx
// FlatList — 大量データの基本形
import { FlatList, Text, View, StyleSheet } from 'react-native';

type Item = { id: string; title: string };

const data: Item[] = Array.from({ length: 1000 }, (_, i) => ({
  id: String(i),
  title: `Item ${i}`,
}));

export function ItemList() {
  return (
    <FlatList
      data={data}                                  // 配列
      renderItem={({ item }) => (                   // 1件をどう描くか
        <View style={styles.row}>
          <Text>{item.title}</Text>
        </View>
      )}
      keyExtractor={(item) => item.id}              // 安定した key（必須級）
    />
  );
}

const styles = StyleSheet.create({
  row: { padding: 16, borderBottomWidth: 1, borderColor: '#eee' },
});
```
```tsx
// SectionList — 見出し付きグループ
import { SectionList, Text } from 'react-native';

const sections = [
  { title: 'A', data: ['Apple', 'Avocado'] },
  { title: 'B', data: ['Banana'] },
];

export function Grouped() {
  return (
    <SectionList
      sections={sections}
      keyExtractor={(item, index) => item + index}
      renderItem={({ item }) => <Text style={{ padding: 12 }}>{item}</Text>}
      renderSectionHeader={({ section }) => (
        <Text style={{ fontWeight: 'bold', padding: 8 }}>{section.title}</Text>
      )}
    />
  );
}
```
```tsx
// ScrollView — 少数・固定の内容だけ（フォーム等）
import { ScrollView, Text } from 'react-native';
export function Settings() {
  return (
    <ScrollView>
      <Text>項目1</Text>
      <Text>項目2</Text>
      {/* 全部一度に描画される。数件ならこれでOK */}
    </ScrollView>
  );
}
```

## 実務での使い方・定番パターン
- **ヘッダー / フッター / 空状態**：`ListHeaderComponent` / `ListFooterComponent` / `ListEmptyComponent` で、検索バー・読み込み中・「データなし」を宣言的に差し込める。
- **無限スクロール**：`onEndReached`（＋ `onEndReachedThreshold`）で末尾近くまで来たら次ページを取得。`refreshing` / `onRefresh` で引っ張って更新（Pull to Refresh）。
- **区切り線**：`ItemSeparatorComponent` で行間の線を統一。`renderItem` 側に書かなくて済む。
- **固定高なら `getItemLayout`**：行の高さが一定なら計算式を渡すと、スクロール位置の算出が速くなり `scrollToIndex` も安定する。
```tsx
<FlatList
  // ...
  getItemLayout={(_, index) => ({ length: 72, offset: 72 * index, index })}
/>
```
- **パフォーマンス調整**：`windowSize`（前後に保持する画面数）/ `initialNumToRender`（初回描画数）/ `maxToRenderPerBatch` を調整。重い行は `renderItem` を `React.memo` 化。
- **横スクロール**：`horizontal` を付ければ横並びのカルーセルにもなる。

## ハマりどころ / アンチパターン
- **大量データを `ScrollView` で出す**：これが最頻の事故。`ScrollView` は全要素をマウントするのでメモリ爆発・初期描画が固まる。**件数が増える可能性があるなら問答無用で `FlatList`**。
- **`keyExtractor` を省く / index を key にする**：並び替え・追加削除で再利用がおかしくなり、表示崩れや無駄な再描画が出る。**一意で安定した id を返す**。
- **`renderItem` 内で毎回新しい関数・オブジェクトを生成**：行ごとに props 参照が変わり再描画が増える。重いリストでは `renderItem` を外出し＋ `React.memo`。
- **可変高で `getItemLayout` を付ける**：高さが行ごとに違うのに固定式を渡すと、スクロール位置がずれる。可変高なら付けない（または計測ライブラリを使う）。
- **`FlatList` を `ScrollView` の中に入れる**：縦スクロールが二重になり、仮想化が効かず警告も出る。入れ子にするなら `ListHeaderComponent` 等で吸収するのが正解。
- **`data` の参照を毎レンダー作り直す**：`data={[...]}` をインラインで作ると毎回別配列扱いで全再描画。`useMemo` か state で安定させる。

## 関連
[core_components.md](./core_components.md) / [components_state.md](./components_state.md)
