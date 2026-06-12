# React Native Testing Library（React Native）

## ひとことで言うと
RN コンポーネントを**仮想的にレンダリング**し、「ユーザーが見えるもの／操作」を起点に検証するライブラリ（RNTL）。中心 API は `render`（描画）/ `screen`（描画結果へのアクセス）/ `fireEvent`（操作の発火）。Web の `@testing-library/react` の RN 版で、`View`/`Text` などネイティブ要素を `getByText` / `getByRole` / `getByTestId` で問い合わせる。テストランナーは RN 同梱の Jest。

## 役割・なぜ必要か
- 実機を起動せず**メモリ上にコンポーネントを描画**して、表示と操作後の変化を高速に検証できる（widget/コンポーネントテストの主役）。
- **実装の内部（state 変数や関数名）ではなく、ユーザーに見える振る舞い**を検証するので、リファクタに強いテストになる。
- `fireEvent.press` などで**タップ・入力を再現**し、UI の分岐（成功/エラー/空）を確実に通せる。
- E2E（Detox/Maestro）より圧倒的に速く安価なので、テストピラミッドの土台を担う。

## 役割・なぜ必要か（補足: 担当範囲）
- RNTL は本物の端末を動かさない。実権限ダイアログ・実機差・タップ座標は **E2E（→ [detox.md](./detox.md)）の担当**。

## 基本の書き方（コード）
```bash
npx expo install --dev jest jest-expo @testing-library/react-native
# package.json: "test": "jest", "jest": { "preset": "jest-expo" }
```
```tsx
// __tests__/Counter.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Counter } from '../Counter';

test('ボタンを押すとカウントが増える', () => {
  // Arrange: 描画
  render(<Counter />);
  expect(screen.getByText('count: 0')).toBeOnTheScreen();

  // Act: 操作を発火
  fireEvent.press(screen.getByText('+1'));

  // Assert: 変化を検証
  expect(screen.getByText('count: 1')).toBeOnTheScreen();
});
```
```tsx
// 入力（changeText）と role/label でのクエリ
import { render, screen, fireEvent } from '@testing-library/react-native';
import { LoginForm } from '../LoginForm';

test('メール入力後にログインできる', () => {
  render(<LoginForm />);
  // ネイティブ要素を「ユーザー視点」で取得
  fireEvent.changeText(screen.getByPlaceholderText('メール'), 'a@example.com');
  fireEvent.press(screen.getByRole('button', { name: 'ログイン' }));
  expect(screen.getByText('ようこそ')).toBeOnTheScreen();
});
```
```tsx
// 非同期：API/状態更新の反映を findBy / waitFor で待つ
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';

test('保存後に「保存しました」が出る', async () => {
  render(<Profile />);
  fireEvent.press(screen.getByText('保存'));
  // findBy = 「待つ getBy」。要素が出るまでリトライ
  expect(await screen.findByText('保存しました')).toBeOnTheScreen();
});
```

## 実務での使い方・定番パターン
- **クエリの優先順位**：`getByText` / `getByRole` / `getByLabelText` / `getByPlaceholderText` を優先し、`getByTestId`（`testID`）は最終手段。ユーザーから見えるもので探すと実装変更に強い。
- **3系統のクエリを使い分け**：要素が**有る前提**は `getBy...`、**無い検証**は `queryBy...`（無ければ null）、**非同期で出る**ものは `findBy...`（Promise）。
- **非同期は `findBy` / `waitFor`**：API・`AsyncStorage`・state 更新の反映を待つ。固定 `sleep` は使わない。
- **AAA で書く**：`render`（Arrange）→ `fireEvent`（Act）→ `getBy/findBy`（Assert）の3段。
- **`renderWithProviders` ヘルパー**：NavigationContainer / Provider（Redux/Zustand/Query）でラップする共通関数を用意すると、画面テストが楽になる。→ [components_state.md](./components_state.md)
- **`jest-native` マッチャ**：`toBeOnTheScreen` / `toHaveTextContent` / `toBeDisabled` 等で可読性を上げる（新しめのRNTLには同梱）。
- **境界だけモック**：ネイティブモジュール（`expo-*`/`react-native-*`）や API は `jest.mock` でモックし、UI ロジック本体は本物のまま検証する。

## ハマりどころ
- **ネイティブモジュールのモック忘れ**：`expo-*` 等はテスト環境に実体が無く `undefined is not a function` で落ちる。`jest.mock` か preset（`jest-expo`）のモックを使う。
- **`act` 警告 / 非同期取りこぼし**：state 更新が描画に反映される前にアサートして失敗する。`findBy...` / `waitFor` で待つ。
- **`getByText` がマッチしない**：表記揺れ（全角/半角・空白）や `<Text>` のネスト分割で落ちる。正規表現マッチや `getByRole` に切り替える。
- **`testID` 頼み**：ID ばかりで検証するとユーザー体験から乖離する。まずテキスト/ロールで探す。
- **`getBy` で「無い」を検証**：要素が無いと `getBy` は例外を投げる。無いことの検証は `queryBy...` + `toBeNull()`。
- **過剰なモック**：内部実装まで縛ると、リファクタで赤くなる「もろい」テストに。境界（API/ストレージ/ネイティブ）だけモックする。
- **E2E と混同**：実権限・実機差・タップ座標は RNTL では再現できない。→ [detox.md](./detox.md)

## 関連
[detox.md](./detox.md) / [testing.md](./testing.md) / [components_state.md](./components_state.md) / [storage_state.md](./storage_state.md)
