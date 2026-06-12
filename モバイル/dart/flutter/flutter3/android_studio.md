# Android Studio（Flutter用）

## ひとことで言うと
Flutter開発に使えるIDEの1つ（**VS Codeと並ぶ二大選択肢**）。**Flutter/Dartプラグイン**を導入して使う。Android周りの機能が手厚いのが持ち味。

## 役割・なぜ必要か
- Flutterは1つのコードでiOS/Android両対応するが、開発環境としてはAndroid Studioか VS Code を選ぶ。Android Studioを選ぶと、**Android SDK・エミュレータ・Gradle**といったAndroid基盤がIDEに統合されていて扱いやすい。
- Android StudioでFlutter開発に効く要素:
  - **Flutter / Dartプラグイン**（補完・**hot reload**・デバッグ・**ウィジェットインスペクタ**）
  - **AVD（Androidエミュレータ）** の作成・起動
  - **Android SDK** の管理（SDK Manager）
  - `android/`（ネイティブAndroid側）の**Gradleビルド**
- `flutter doctor` がAndroid Studio本体やSDKを自動検出してくれるため、環境が整っているかをコマンド一発で確認できる。

## 基本の使い方
事前準備（プラグイン導入後）:
```bash
# 環境チェック（AS・SDK・接続端末を検出）
flutter doctor -v

# 利用可能なデバイス確認
flutter devices
```

開発の基本フロー:
```bash
flutter create my_app      # プロジェクト作成
cd my_app
flutter run                # 実行（保存でhot reload）
```

Android Studio上では:
- ツールバーのデバイス選択でエミュレータ/実機を切り替え
- 実行後、コード保存またはツールバーのボタンで **hot reload**（状態を保ったままUI更新）
- **Flutter Inspector** でウィジェットツリーを可視化し、レイアウト崩れを調査

```dart
// hot reloadの恩恵が分かる最小例
class Hello extends StatelessWidget {
  @override
  Widget build(BuildContext context) =>
      const Text('Hello'); // 'Hi'に変えて保存 → 即反映
}
```

## 実務での勘所
- **VS Codeも人気**（軽い・起動が速い）。AndroidまわりはAndroid Studioが手厚いので、「Android中心ならAS、軽快さ重視ならVS Code」で選ぶとよい。両方入れて使い分ける人も多い。
- **iOSビルドはMac + Xcode必須**。Android StudioではiOSの実機/シミュレータ向けビルドや署名はできない。Flutterでも iOS は Xcode から逃れられない。
- `flutter doctor` の出力に出る警告（未導入プラグイン、ライセンス未同意など）を**全部緑にしてから**開発を始めると詰まりにくい。
- hot reload で反映されない変更（`main()`変更・グローバル状態・ネイティブ側）は **hot restart**（再起動）が必要、という違いを把握しておく。

## ハマりどころ
- **Flutter/Dartプラグイン未導入**: これを入れないとAndroid Studioは普通のAndroid用IDEのまま。`Settings > Plugins` から Flutter（Dartも自動追加）を入れる。
- **Android SDKライセンス未同意**: `flutter doctor` で赤くなる定番。次で一括同意する。
  ```bash
  flutter doctor --android-licenses
  ```
- **AVD（エミュレータ）**: 未作成だと実行先がなく、リソース消費も大きい。実機デバッグの方が軽いことも。
- **iOSはXcode必須**: Android StudioだけでiOSを動かそうとして詰まるのは初心者の典型。iOSビルド時はMacに切り替える。
- Flutter SDK と Dart SDK のバージョン、`android/` 側のGradle/AGPの互換に注意。`flutter upgrade` 後にAndroidビルドが落ちたら、まずGradle周りのバージョンを疑う。

## 関連
[getting_started.md](./getting_started.md)
