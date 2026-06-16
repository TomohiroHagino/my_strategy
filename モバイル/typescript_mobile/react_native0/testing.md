# テスト（Jest / RN Testing Library）（React Native）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。RN の定番は **Jest**（テストランナー）＋ **React Native Testing Library（RNTL）** で、ユニットテストとコンポーネントテストを書く。実機操作を再現する E2E は **Detox / Maestro**（まったく別の仕組み）。

## 役割・なぜ必要か
- 変更のたびに手で全画面を確認するのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- **Jest**：テストの実行・アサーション・モックを担うランナー。RN テンプレートに同梱。
- **RNTL**：コンポーネントを**仮想的にレンダリング**し、「ユーザーが見えるもの／操作」を起点に検証する（実装の内部ではなく振る舞いを見る）。中心 API は `render` / `screen` / `fireEvent`。
- **E2E（Detox / Maestro）** は別レイヤー：実機・シミュレータ上で**本物のアプリを起動**してタップ・入力する。ユニット/コンポーネントとは目的も実行環境も違う。

## 基本の書き方（コード）
```bash
npx expo install --dev jest jest-expo @testing-library/react-native
# package.json: "test": "jest", "jest": { "preset": "jest-expo" }
```
```tsx
// __tests__/Counter.test.tsx（コンポーネントテスト）
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Counter } from '../Counter';

test('ボタンを押すとカウントが増える', () => {
  // Arrange
  render(<Counter />);
  expect(screen.getByText('count: 0')).toBeTruthy();

  // Act
  fireEvent.press(screen.getByText('+1'));

  // Assert
  expect(screen.getByText('count: 1')).toBeTruthy();
});
```
```tsx
// 非同期：保存後に表示が変わるのを waitFor で待つ
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { Profile } from '../Profile';

test('保存ボタンで「保存しました」が出る', async () => {
  render(<Profile />);
  fireEvent.press(screen.getByText('保存'));
  await waitFor(() => {
    expect(screen.getByText('保存しました')).toBeTruthy(); // 表示されるまで待つ
  });
});
```
```ts
// ネイティブモジュールはモックする（テスト環境に実体が無いため）
jest.mock('expo-secure-store', () => ({
  setItemAsync: jest.fn(),
  getItemAsync: jest.fn().mockResolvedValue('dummy-token'),
}));
```

## 実務での使い方・定番パターン
- **テストピラミッド**を意識：土台に多数の高速な **ユニット/コンポーネントテスト**（Jest + RNTL）、頂点に少数の **E2E**（Detox/Maestro）を重要フローだけ。
- **クエリは「ユーザー視点」で**：`getByText` / `getByRole` / `getByLabelText` を優先し、`testID`（`getByTestId`）は最終手段。これで実装変更に強いテストになる。
- **AAA で書く**：Arrange（`render`）→ Act（`fireEvent`）→ Assert（`getBy...`）の3段で読みやすく。
- **非同期は `waitFor` / `findBy...`**：API・`AsyncStorage`・状態更新の反映を待つ。`findByText` は「待つ getBy」。→ [storage_state.md](./storage_state.md)
- **E2E は別ツールで別運用**：**Maestro** は YAML で手軽、**Detox** はグレーボックスで安定待機が強い。CI ではビルド済みアプリを実機/シミュレータで動かす。
- **ナビゲーションを含むテスト**：NavigationContainer でラップして描画する。共通の `renderWithProviders` ヘルパーを用意すると楽。→ [components_state.md](./components_state.md)

## ハマりどころ / アンチパターン
- **ネイティブモジュールのモック忘れ**：`expo-*` や `react-native-*` はテスト環境に実体が無く `undefined is not a function` で落ちる。`jest.mock` でモックするか、preset（`jest-expo`）のモックを使う。
- **`act` 警告 / 非同期の取りこぼし**：状態更新が描画に反映される前にアサートして失敗する。**`waitFor` / `findBy...`** で待つ。`sleep` 的な固定待ちはしない。
- **E2E をユニットと混同**：RNTL は本物の端末を動かさない。タップ位置・実権限ダイアログ・実機差は **E2E（実機/シミュレータ）が必須**。逆に E2E で細かいロジックを総当たりすると遅く不安定。
- **過剰なモック**：内部実装まで縛ると、リファクタで赤くなる「もろい」テストに。境界（API・ストレージ・ネイティブ）だけモックする。
- **`testID` 頼み**：見た目のラベルでなく ID ばかりで検証すると、ユーザー体験から乖離する。まずテキスト/ロールで探す。
- **flaky な E2E**：固定 `sleep` 依存で不安定化。Detox/Maestro の**自動待機**を使い、ネットワークはモックサーバで安定させる。
- **カバレッジ偏重**：数字（80%目安）だけ追って**重要フローの E2E が無い**のは本末転倒。

## 関連
[components_state.md](./components_state.md) / [storage_state.md](./storage_state.md) / [native_modules.md](./native_modules.md)
