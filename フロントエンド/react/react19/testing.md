# テスト（Testing）（React）

## ひとことで言うと
コンポーネントの**振る舞いを自動検証するコード**。React の定番は **React Testing Library（RTL）**＝「ユーザが見て・操作する視点」でテストする方針のライブラリ。実行基盤（test runner）は **Vitest**（Vite系で主流）か **Jest**。RTL は「どう実装したか」ではなく「画面に何が出て、操作すると何が起きるか」を検証する。

## 役割・なぜ必要か
- 手で全画面を毎回確認するのは非現実的。**回帰（デグレ）を自動検出**するために要る。
- RTL は **アクセシビリティ的な見え方**（role / label / text）でDOMを探すので、テストが実装の内部構造に依存しにくく、リファクタに強い。
- 「この操作でこう動く」という実行可能な仕様書になり、設計変更の安全網になる。

## 基本の書き方（コード）
`render`（描画）→ `screen`（探す）→ `userEvent`（操作）→ `expect`（検証）の流れ。
```tsx
// Counter.test.tsx（Vitest + React Testing Library）
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { expect, test } from "vitest";
import { Counter } from "./Counter";

test("ボタンを押すとカウントが増える", async () => {
  // Arrange：描画
  render(<Counter />);
  const button = screen.getByRole("button", { name: "増やす" });

  // Act：ユーザ操作（クリック）
  await userEvent.click(button);

  // Assert：振る舞いを検証
  expect(screen.getByText("カウント: 1")).toBeInTheDocument();
});
```
要素の探し方は**ユーザに近い順**で選ぶ：`getByRole` ＞ `getByLabelText`（フォーム）＞ `getByText` ＞（最後の手段）`getByTestId`。

## 実務での使い方・定番パターン
- **クエリの種類を使い分け**：
  - `getBy*`：今すぐ在るはず（無ければ即エラー）。
  - `queryBy*`：**無いこと**の検証用（`expect(...).not.toBeInTheDocument()`）。
  - `findBy*`：**非同期で後から現れる**要素を待つ（Promise を返す。`await` する）。
- **非同期の待ち**：データ取得後の表示は `findBy` か `waitFor` で待つ。`sleep` 的な固定待ちはしない。
```tsx
test("取得したユーザ名が表示される", async () => {
  render(<UserCard id="1" />);
  // ローディングが消えて名前が出るのを待つ
  expect(await screen.findByText("Taro")).toBeInTheDocument();
});
```
- **API はモック**：`msw`（Mock Service Worker）でネットワーク層をモックすると、実装に近い形で安定。`fetch` 直モックより壊れにくい。
- **フォーム**：`userEvent.type` で入力、`getByLabelText` で `<label>` 経由に取得（アクセシブルな実装の担保にもなる）。送信後のバリデーションメッセージを `findByText` で待つ。
- **Provider が要るコンポーネント**：QueryClientProvider / Router / Context を包む `renderWithProviders` ヘルパーを用意して使い回す（DRY）。
- **テストピラミッド**：土台に多数の高速なユニット/コンポーネントテスト、頂点に少数の E2E（Playwright 等で重要フローのみ）。視覚回帰はスクリーンショットで補完。

## ハマりどころ / アンチパターン
- **実装詳細をテストする**：state の中身や特定クラス名、子コンポーネントの呼ばれ方を検証すると、リファクタで赤くなる「もろい」テストに。**ユーザに見える結果**（テキスト・role・有効/無効）を検証する。`getByTestId` 多用は実装依存のサイン。
- **`act(...)` 警告**：状態更新が `await` されていないと出る。`userEvent` は内部で `act` を処理するので **`await userEvent.xxx()` を必ず await**。非同期表示は `findBy`/`waitFor` で包む。
- **非同期待ちのミス**：取得前に `getBy` で探して落ちる。後から出るものは `findBy`、消えるものは `waitFor(() => expect(...).not...)`。
- **`waitFor` の中で副作用**：`waitFor` のコールバックには**アサーションだけ**。中で `userEvent` を呼ばない（何度も再実行されるため）。
- **過剰モック**：実装の内部呼び出しまでモックすると壊れやすい。境界（HTTP）だけ `msw` でモック。
- **テスト間の状態リーク**：`afterEach(cleanup)`（多くの設定で自動）と、モック/タイマーのリセットを忘れると順序依存で落ちる。
- **カバレッジ偏重**：数字（80%目安）だけ追って**重要フローのテストが無い**のは本末転倒。

## 関連
[forms.md](./forms.md) / [data_fetching.md](./data_fetching.md) / [hooks.md](./hooks.md)
