# ネイティブ連携（New Architecture / Expo modules）（React Native）

## ひとことで言うと
JavaScript から OS のネイティブ機能（カメラ・位置情報・センサー・暗号化など）を呼び出す仕組み。RN 0.7x の標準は **New Architecture（JSI / Fabric / TurboModules）**で、実務ではほぼ **Expo modules / コミュニティパッケージ**を入れて使う。自前のネイティブモジュールを書く場面は稀。

## 役割・なぜ必要か
- JS だけでは触れない OS の機能（Bluetooth・生体認証・プッシュ通知など）に橋渡しするのがネイティブ連携。
- **New Architecture** はその橋渡しを高速・型安全にした新しい標準。中核は3つ。
  - **JSI（JavaScript Interface）**：JS とネイティブを **C++ で直接** つなぐ層。旧来の JSON シリアライズ越しの「ブリッジ」を置き換え、同期呼び出しも可能に。
  - **Fabric**：新しい **UI レンダラ**。`<View>` などのネイティブUIをJSIベースで描画し、レイアウトとイベントを高速化。
  - **TurboModules**：ネイティブ機能（カメラ等）を呼ぶ **新しいモジュール方式**。必要になった時に**遅延ロード**され、型は Codegen で生成される。
- 旧 **Bridge** との違い：旧来は JS ⇄ ネイティブを **非同期・JSON文字列** でやり取りしていた（遅い・型が曖昧）。New Arch は **JSI 経由で直接参照**するため速く、型も安全。RN 0.7x では New Arch が既定方向。

## 基本の書き方（コード）
```bash
# ほぼ「パッケージを入れて使う」だけ。自前ネイティブは基本書かない。
npx expo install expo-camera expo-location expo-secure-store

# ネイティブ依存を含むため Expo Go では動かない → 開発ビルドを作る
npx expo install expo-dev-client
eas build --profile development --platform ios      # 実機/シミュレータ用
eas build --profile development --platform android
```
```tsx
// カメラ権限を取り、撮影する例（Expo modules を「ただ呼ぶ」）
import { useState } from 'react';
import { Button, Text, View } from 'react-native';
import { CameraView, useCameraPermissions } from 'expo-camera';

export function CameraScreen() {
  const [permission, requestPermission] = useCameraPermissions();
  const [ready, setReady] = useState(false);

  if (!permission) return <Text>権限を確認中…</Text>;
  if (!permission.granted) {
    return (
      <View>
        <Text>カメラの権限が必要です</Text>
        <Button title="許可する" onPress={requestPermission} />
      </View>
    );
  }
  return <CameraView style={{ flex: 1 }} onCameraReady={() => setReady(true)} />;
}
```

## 実務での使い方・定番パターン
- **まず既製パッケージを探す**：カメラ/位置/通知/生体認証などは **Expo modules** か **コミュニティ製**（`react-native-*`）でほぼ揃う。自前 TurboModule は最後の手段。
- **インストールは `npx expo install`** を使う（`npm install` ではなく）。RN のバージョンに合う互換版を選んでくれる。
- **prebuild / dev build を運用に組み込む**：ネイティブ依存が入った瞬間、素の Expo Go では動かなくなる。`expo-dev-client` ＋ `eas build`（または `npx expo prebuild` でネイティブ生成）に切り替える。→ [getting_started.md](./getting_started.md)
- **権限まわりは設定ファイルに宣言**：`app.json` / `app.config.ts` の plugins で iOS の `Info.plist`・Android の権限を自動注入。手で `Info.plist` を編集しないのが Expo 流。→ [platform.md](./platform.md)
- **New Architecture の有効化**：RN 0.7x では `newArchEnabled` を `app.json` で有効化（テンプレートにより既定）。使うパッケージが New Arch 対応かは必ず確認。
- **自前ネイティブが本当に要るとき**だけ Expo Modules API（Swift/Kotlin）で薄く書き、Config Plugin でビルドに組み込む。生 Objective-C/Java を直接書く場面はさらに稀。

## ハマりどころ / アンチパターン
- **「Expo Go で動かない」の正体**：ネイティブ依存（カメラ・生体認証など）は Expo Go アプリにバンドルされていない。**dev build（開発ビルド）/ prebuild が必要**。ここを知らずに「ライブラリが壊れてる」と誤解しがち。
- **`npm install` で入れて互換崩れ**：RN バージョン非対応の版が入りクラッシュ。`npx expo install` を使う。
- **New Architecture 非対応パッケージ**：古いライブラリが New Arch で動かない/警告。対応状況（`react-native-new-architecture` の対応表）を確認し、未対応なら代替を探す。
- **iOS だけ/Android だけ動く**：権限宣言漏れ（iOS は `Info.plist` の用途文字列必須、Android は `AndroidManifest`）。Config Plugin / `app.json` に両 OS 分を書く。両プラットフォームで実機確認する。
- **ビルド環境の沼**：iOS は macOS + Xcode、Android は JDK/SDK が要る。CI では **EAS Build**（クラウドビルド）に寄せるとローカル差異を避けやすい。
- **同期 JSI を過信**：JSI で同期呼び出しができるからと重い処理を同期で呼ぶと UI が固まる。重い処理は非同期/別スレッドへ。

## 関連
[getting_started.md](./getting_started.md) / [platform.md](./platform.md) / [storage_state.md](./storage_state.md)
