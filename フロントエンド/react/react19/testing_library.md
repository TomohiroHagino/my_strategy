# React Testing Library（React）

## ひとことで言うと
React コンポーネントを **ユーザー視点（画面に出るテキスト・role・操作）で検証する**ライブラリ。`render` で描画し、`screen.getByRole` などで要素を探し、`userEvent` で操作して `expect` で検証する。実装の内部構造（state・クラス名）に依存しないテストを書くための方針が組み込まれている。

## 役割・なぜ必要か
- 「どう実装したか」ではなく「ユーザーに何が見えて、操作すると何が起きるか」を検証する。実装をリファクタしてもテストが壊れにくい。
- 要素探索が **アクセシビリティの見え方**（role / label / text）ベースなので、テストを書くこと自体がアクセシブルな実装の担保になる。
- 手で全画面を毎回確認するのは非現実的。回帰（デグレ）を自動検出する。
- ランナー（Vitest / Jest）とは別レイヤー。RTL は「探す・操作する・検証する」を担当する。

## 基本の書き方（コード）
`render`（描画）→ `screen`（探す）→ `userEvent`（操作）→ `expect`（検証）。
```tsx
// Counter.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { expect, test } from "vitest";
import { Counter } from "./Counter";

test("ボタンを押すとカウントが増える", async () => {
  render(<Counter />);
  const button = screen.getByRole("button", { name: "増やす" });

  await userEvent.click(button); // userEvent は await 必須

  expect(screen.getByText("カウント: 1")).toBeInTheDocument();
});
```
要素の探し方は **ユーザーに近い順** で選ぶ：`getByRole` ＞ `getByLabelText`（フォーム）＞ `getByText` ＞（最後の手段）`getByTestId`。

## 実務での使い方・定番パターン
- **クエリの使い分け**：
  - `getBy*`：今すぐ在るはず（無ければ即エラー）。
  - `queryBy*`：**無いこと**の検証用（`expect(...).not.toBeInTheDocument()`）。
  - `findBy*`：**非同期で後から現れる**要素を待つ（Promise を返す。`await` する）。
```tsx
test("取得したユーザー名が表示される", async () => {
  render(<UserCard id="1" />);
  expect(await screen.findByText("Taro")).toBeInTheDocument(); // 出るまで待つ
});
```
- **フォーム**：`userEvent.type` で入力、`getByLabelText` で `<label>` 経由に取得。送信後のバリデーションメッセージは `findByText` で待つ。
- **Provider が要るコンポーネント**：QueryClientProvider / Router / Context を包む `renderWithProviders` ヘルパーを用意して使い回す（DRY）。
```tsx
function renderWithProviders(ui: React.ReactElement) {
  const client = new QueryClient();
  return render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}
```
- **API は境界でモック**：`fetch` を直接モックするより、ネットワーク層を `msw` で差し替えると実装に近く壊れにくい。→ [msw.md](./msw.md)

## ハマりどころ
- **`act(...)` 警告**：状態更新が `await` されていないと出る。`userEvent` は内部で `act` を処理するので **`await userEvent.xxx()` を必ず await**。非同期表示は `findBy`/`waitFor` で包む。
- **非同期待ちのミス**：取得前に `getBy` で探して落ちる。後から出るものは `findBy`、消えるものは `waitFor(() => expect(...).not...)`。
- **`waitFor` の中で副作用**：`waitFor` のコールバックには **アサーションだけ**。中で `userEvent` を呼ばない（何度も再実行されるため）。
- **`getByTestId` 多用**：実装依存のサイン。`role`/`label`/`text` で取れないか先に検討する。
- **テスト間の状態リーク**：`afterEach(cleanup)`（多くの設定で自動）とモック/タイマーのリセットを忘れると順序依存で落ちる。

## 関連
[testing.md](./testing.md)（全体方針・ランナー） / [msw.md](./msw.md) / [playwright.md](./playwright.md) / [forms.md](./forms.md)
