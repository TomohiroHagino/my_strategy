# プラットフォーム差分 / 権限（React Native）

## ひとことで言うと
1つのコードベースで iOS と Android の両方を動かすRNで、**OSごとの違いを吸収する仕組み**（`Platform.OS` / `Platform.select` / プラットフォーム別ファイル）と、カメラ・位置情報などを使う前に必要な**権限リクエスト**、そしてノッチや切り欠きを避ける **`SafeAreaView`** をまとめた話。

## 役割・なぜ必要か
- 「Write once, run anywhere」とはいえ、UIの細部・キーボード挙動・権限の流れは **iOSとAndroidで違う**。その差分を安全に分岐するための公式APIが `Platform`。
- スマホはユーザーのプライバシー資産（カメラ・連絡先・位置）に触れるため、**OSが明示的な権限許可を要求する**。許可前に使うとクラッシュ／無音で失敗する。
- iPhoneのノッチやAndroidのステータスバーに**UIが食い込む**のを防ぐため、安全領域（Safe Area）の概念が必要。

## 基本の書き方（コード）
```tsx
import { Platform, StyleSheet, View, Text } from 'react-native';

// ① OS で分岐
if (Platform.OS === 'ios') {
  // iOS だけの処理
}

// ② スタイルや値を OS で出し分け
const styles = StyleSheet.create({
  header: {
    paddingTop: Platform.select({ ios: 44, android: 24, default: 0 }),
    fontFamily: Platform.select({ ios: 'Helvetica', android: 'Roboto' }),
  },
});

// ③ バージョン分岐も可能
const isModernAndroid = Platform.OS === 'android' && Platform.Version >= 33;

export function Header() {
  return (
    <View style={styles.header}>
      <Text>実行中: {Platform.OS}</Text>
    </View>
  );
}
```

```
// ④ プラットフォーム別ファイル（import 名は拡張子なしで書く）
//   Button.ios.tsx     ← iOS でだけ採用される
//   Button.android.tsx ← Android でだけ採用される
//   import { Button } from './Button';  ← バンドラが自動で振り分け
```

## 実務での使い方・定番パターン
- **`SafeAreaView`** でノッチ・ステータスバー・ホームインジケータを避ける。実務では `react-native-safe-area-context` の `SafeAreaView` / `useSafeAreaInsets` を使うのが定番（コアのものより柔軟で、Androidも安定）。
```tsx
import { SafeAreaView } from 'react-native-safe-area-context';
export function Screen() {
  return (
    <SafeAreaView style={{ flex: 1 }}>
      {/* ノッチに食い込まない安全領域に描画される */}
    </SafeAreaView>
  );
}
```
- **権限（Expo）** は機能ライブラリ側が要求関数を持つ。使う直前に許可を取り、結果で分岐する。
```tsx
import { useCameraPermissions } from 'expo-camera';
function CameraButton() {
  const [permission, requestPermission] = useCameraPermissions();
  if (!permission?.granted) {
    return <Button title="カメラを許可" onPress={requestPermission} />;
  }
  // granted のときだけカメラ機能を出す
}
```
- 位置情報は `expo-location` の `requestForegroundPermissionsAsync()`、通知は `expo-notifications` …とライブラリごとに専用関数がある。**`app.json` / `Info.plist` に利用理由（usage description）の文言登録も必須**（特にiOS）。
- 小さなOS差分はインラインの `Platform.select` で十分。**分岐が大きく増えたら `.ios.tsx` / `.android.tsx` にファイルごと分ける**と読みやすい。

## ハマりどころ / アンチパターン
- **OS差分の見落とし**。iOSで作って満足し、Androidで `KeyboardAvoidingView`・影（iOSは`shadow*`、Androidは`elevation`）・フォントが崩れる。**必ず両OSの実機/エミュレータで確認**。
- **権限リクエストの流れを誤解**。「許可ダイアログは1回だけ」「ユーザーが拒否したら次回から自動では出ない（設定アプリへ誘導が必要）」。`granted` を毎回チェックし、拒否時のフォールバックUIを用意する。
- **使用理由（usage description）の未登録**でiOSがクラッシュ／審査リジェクト。`NSCameraUsageDescription` 等を `app.json`（Expo）で必ず設定。
- **`SafeAreaView` 未対応**でノッチや下部バーにボタンが隠れる／ステータスバーに文字が重なる。`flex: 1` を付け忘れて領域が潰れるのも頻出。
- **`SafeAreaView` をスクロール内側に置く**と効かない。原則は**画面のルート**に置く。
- **`Platform.select` で `default` を書かない** → Web（react-native-web）等で `undefined` になり崩れる。
- **OS判定でロジックを増やしすぎる**。差分が散らばるなら**プラットフォーム別ファイル**に切り出す方が保守しやすい。

## 関連: [native_modules.md](./native_modules.md)
