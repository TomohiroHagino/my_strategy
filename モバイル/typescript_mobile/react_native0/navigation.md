# ナビゲーション（React Native）

## ひとことで言うと
画面（Screen）間の遷移を担う仕組み。RN にはルーティングが標準で無いので、デファクトの **React Navigation**（`@react-navigation/native`）か、Expo の **Expo Router**（ファイルベース）を使う。Web の `react-router` に相当するレイヤ。

## 役割・なぜ必要か
- モバイルは「スタックで積む／戻る」「下タブで切り替える」「横からドロワーを出す」という**ネイティブらしい遷移**が必要。これを宣言的に組むのが Navigator。
- 画面間で値を渡す（`params`）、ヘッダーや戻るボタンを自動で出す、ディープリンク（URL から特定画面を開く）を扱う、といった面倒事をまとめて引き受けてくれる。
- React Navigation は「**Navigator（入れ物）＋ Screen（中身）**」の階層構造で画面を定義する。Expo Router は「**ファイル配置＝ルート**」で同じことをやる。

## 基本の書き方（コード）
```tsx
// App.tsx — React Navigation（Stack）
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

// 各画面に渡る params の型を一元定義（型付けの肝）
export type RootStackParamList = {
  Home: undefined;
  Detail: { id: string };   // Detail は id を必須で受け取る
};

const Stack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Detail" component={DetailScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```
```tsx
// screens/HomeScreen.tsx — 遷移する側
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import { View, Button } from 'react-native';
import type { RootStackParamList } from '../App';

type Props = NativeStackScreenProps<RootStackParamList, 'Home'>;

export function HomeScreen({ navigation }: Props) {
  return (
    <View>
      {/* navigate(画面名, params) で遷移 */}
      <Button title="詳細へ" onPress={() => navigation.navigate('Detail', { id: '42' })} />
    </View>
  );
}
```
```tsx
// screens/DetailScreen.tsx — 受け取る側
import { Text } from 'react-native';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import type { RootStackParamList } from '../App';

type Props = NativeStackScreenProps<RootStackParamList, 'Detail'>;

export function DetailScreen({ route }: Props) {
  const { id } = route.params;        // route.params で受け取る（型付き）
  return <Text>id = {id}</Text>;
}
```

## 実務での使い方・定番パターン
- **Tab / Drawer を組み合わせる**：下タブ（`@react-navigation/bottom-tabs`）の各タブの中に Stack を入れ子にするのが定番。タブを跨いでも各タブの履歴が保持される。
```tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
const Tab = createBottomTabNavigator();
// <Tab.Navigator> の <Tab.Screen> に Stack Navigator を component として渡す
```
- **ネストした型付け遷移**：`navigation.navigate('Tabs', { screen: 'Profile', params: { id } })` のように親 Navigator 経由で子 Screen を指定できる。
- **Expo Router（ファイルベース）**：`app/` ディレクトリのファイル配置がそのままルートになる。`app/index.tsx` → `/`、`app/detail/[id].tsx` → `/detail/:id`。遷移は `useRouter().push('/detail/42')`、受け取りは `useLocalSearchParams()`。Web の Next.js App Router に近い感覚。
```tsx
// app/detail/[id].tsx — Expo Router
import { useLocalSearchParams } from 'expo-router';
import { Text } from 'react-native';
export default function Detail() {
  const { id } = useLocalSearchParams<{ id: string }>();
  return <Text>id = {id}</Text>;
}
```
- **戻る・リセット**：`navigation.goBack()` / `navigation.popToTop()` / `navigation.reset()`（ログイン後にスタックを作り直す等）。

## ハマりどころ / アンチパターン
- **Navigator の階層構造を見失う**：Stack の中に Tab、Tab の中にまた Stack…と入れ子が深くなると `navigate` のパス指定がややこしくなる。**画面ツリーを図に書いてから組む**のが結局速い。
- **`params` の型付けを怠る**：`ParamList` 型を作らずに `route.params` を `any` で触ると、渡し忘れ・スペルミスがランタイムまで出ない。**`RootStackParamList` を一元管理**し、各 Screen で `ScreenProps<List, 'Name'>` を使う。
- **`<NavigationContainer>` の二重ラップ**：これはアプリ全体で1つだけ。子の Navigator はラップしない。
- **ディープリンク設定漏れ**：`linking` 設定（`prefixes` / `config`）を入れないと URL から画面を開けない。Expo Router はファイル構造から自動生成されるが、React Navigation は手で `linking` を書く必要がある。
- **画面外で `navigation` を使いたい**：コンポーネント外（通知ハンドラ等）からは props が無いので、`navigationRef`（`createNavigationContainerRef`）を使う。
- **`navigate` と `push` の違い**：同じ画面へ重ねたいときは `push`（`navigate` は同名画面が既にあると再利用してしまう）。

## 関連
[components_state.md](./components_state.md) / [core_components.md](./core_components.md)
