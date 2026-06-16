# コンポーネントと状態（React Native）

## ひとことで言うと
**React そのまま**。関数コンポーネント・hooks（`useState` / `useEffect`）・props / state・JSX、全部 Web の React と同じ。違うのは **JSX に書く UI タグ（`<div>` ではなく `<View>` / `<Text>`）** と、**ブラウザ API（`window` / `document` / `localStorage` など）が存在しない**点だけ。

## 役割・なぜ必要か
- 画面を「**状態（state）→ 表示（JSX）**」の関数として書くのが React の核。RN でもこの考え方は1ミリも変わらない。
- props で親から値を受け取り、state で自分の中の変化を持ち、変わったら再レンダー。この一方向データフローはネイティブでも同じ。
- だから React を知っているなら新しく覚えることはほぼ無い。**「タグが違うだけ」** と割り切ってよい。React の基礎（再レンダー・依存配列・key など）は → [../../../フロントエンド/react/react19/](../../../フロントエンド/react/react19/) を参照。

## 基本の書き方（コード）
```tsx
// components/Counter.tsx
import { useState, useEffect } from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';

type Props = {
  label: string;        // 親から受け取る props（Web と同じ）
  initial?: number;
};

export function Counter({ label, initial = 0 }: Props) {
  // state（useState）も Web と完全に同じ
  const [count, setCount] = useState(initial);

  // 副作用（useEffect）も同じ。依存配列の挙動も同じ。
  useEffect(() => {
    console.log(`count changed: ${count}`);
    // クリーンアップも同じ書き方
    return () => console.log('cleanup');
  }, [count]);

  // 違いはここ：<div> ではなく <View>、文字は必ず <Text> の中
  return (
    <View style={styles.box}>
      <Text style={styles.label}>{label}: {count}</Text>
      <Button title="+1" onPress={() => setCount((c) => c + 1)} />
    </View>
  );
}

const styles = StyleSheet.create({
  box: { padding: 16, gap: 8 },
  label: { fontSize: 18 },
});
```
ポイントは「`useState` / `useEffect` / props の型付け / 関数更新 `setCount(c => c+1)`」が **Web の React コードそのまま** だということ。

## 実務での使い方・定番パターン
- **カスタムフックで共通化**：`useXxx` を切り出して状態ロジックを再利用。Web と同じ作法でそのまま動く。
```tsx
// hooks/useToggle.ts
import { useState, useCallback } from 'react';

export function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn((v) => !v), []);
  return { on, toggle } as const;
}
```
- **`onPress` がクリックの代わり**：Web の `onClick` は無い。タップ系は `onPress`（`Button` / `Pressable` / `TouchableOpacity`）。これは UI タグの話で、state 管理とは別レイヤ。
- **派生値は state にしない**：`count` から計算できるものは `useMemo` か単なる変数で。Web と同じく「state は最小限」。
- **永続化はブラウザ API ではなく専用ライブラリ**：`localStorage` が無いので、保存は `AsyncStorage` 等を使う。→ [storage_state.md](./storage_state.md)
- **イベント・タイマー**：`setTimeout` / `setInterval` は使える（JS 標準なので）。`useEffect` のクリーンアップで `clearTimeout` するのも Web と同じ。

## ハマりどころ / アンチパターン
- **ブラウザ API は無い**：`window` / `document` / `localStorage` / `navigator`（の DOM 部分）/ `alert()` などはそのまま使うと **`undefined` で落ちる**。
  - `localStorage` → `@react-native-async-storage/async-storage`（→ [storage_state.md](./storage_state.md)）
  - `alert()` → RN の `Alert.alert()`
  - `window.innerWidth` → `Dimensions.get('window')` や `useWindowDimensions()`
  - `document.cookie` / DOM 操作系 → そもそも DOM が無いので使えない
- **`<View>` の中に生テキストを置くと落ちる**：文字は必ず `<Text>` で包む。これは UI タグ側の制約（→ [core_components.md](./core_components.md)）で、React の作法とは別問題。
- **`className` ではなく `style`**：CSS クラスは無い。スタイルはオブジェクト。ただしこれもタグ/スタイル層の話で、state 管理は無関係。
- **「RN だから特別なフックがいる」と誤解**：基本の状態管理に RN 専用フックは要らない。`useState` / `useEffect` / `useMemo` / `useCallback` / `useRef` はすべて `react` から import（`react-native` からではない）。
- **再レンダーの最適化も Web と同じ**：不要再レンダーは `React.memo` / `useCallback` で対処。RN 特有の魔法は無い。

## 関連
[core_components.md](./core_components.md) / [storage_state.md](./storage_state.md) / [../../../フロントエンド/react/react19/](../../../フロントエンド/react/react19/)
