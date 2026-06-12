# React Testing Library（Next.js 15）

## ひとことで言うと
React コンポーネントを **ユーザー視点（画面に出るテキスト・role・操作）で検証する**ライブラリ。Next.js 15（App Router）では **Client Component は RTL でそのままテストできる**が、**Server Component（async・RSC）は RTL の `render` では実行できない**ため、ロジックの単体テストや E2E（Playwright）に切り分ける、という線引きが要点になる。

## 役割・なぜ必要か
- 「どう実装したか」ではなく「ユーザーに何が見えて、操作すると何が起きるか」を検証する。リファクタに強い。
- App Router では描画の境界が **サーバ / クライアント** に分かれる。RTL が担当できるのは **クライアント側の対話** であり、ここを明確にすると無駄なテストや失敗を避けられる。
- 手動リグレッションを自動化し、回帰を検出する。
- ランナー（Vitest / Jest）とは別レイヤー。RTL は「探す・操作する・検証する」を担当する。

## 基本の書き方（コード）
Client Component を `render` → `screen` で探す → `userEvent` で操作 → `expect`。
```tsx
// Counter.tsx は "use client"
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { expect, test } from "vitest";
import { Counter } from "./Counter";

test("ボタンを押すとカウントが増える", async () => {
  render(<Counter />);
  await userEvent.click(screen.getByRole("button", { name: "増やす" }));
  expect(screen.getByText("カウント: 1")).toBeInTheDocument();
});
```

## 実務での使い方・定番パターン
- **Client Component → RTL で直接テスト**：`"use client"` の対話コンポーネントは React と同じ要領。`getByRole` ＞ `getByLabelText` ＞ `getByText` の順で探す。
- **Server Component（async）→ 分割する**：
  - データ整形などの **ロジックは純関数に切り出して単体テスト**。
  - 「サーバで取得して描画して表示される」までの確認は **Playwright で E2E**。→ [playwright.md](./playwright.md)
```tsx
// async Server Component を RTL の render に渡すのは不可。ロジックを抜き出す
export function buildTitle(user: { name: string }) {
  return `${user.name} さんのページ`;
}
// title.test.ts
expect(buildTitle({ name: "Taro" })).toBe("Taro さんのページ");
```
- **`next/navigation` のモック**：`useRouter` / `usePathname` 等を使う Client Component は、これらを `vi.mock("next/navigation", ...)` でモックして単体テストする。
- **API は境界でモック**：`fetch` を直接モックするより MSW でネットワーク層を差し替える方が壊れにくい（クライアント取得時）。
- **Server Actions**：呼び出される関数の入出力を単体テストし、フォーム送信からの一連は E2E で確認する。

## ハマりどころ
- **Server Component を `render` しようとして失敗**：async コンポーネントは RTL では描画できない。ロジック分離 ＋ E2E に切り替える。
- **`next/navigation` 未モックでエラー**：テスト環境にはルーターが無い。`useRouter` 等は明示的にモックする。
- **`act(...)` 警告**：`await userEvent.xxx()` を必ず await。非同期表示は `findBy`/`waitFor` で包む。
- **`getByTestId` 多用**：実装依存のサイン。`role`/`label`/`text` を先に検討。
- **テスト間の状態リーク**：`afterEach(cleanup)` とモック/ルーターモックのリセットを忘れると順序依存で落ちる。

## 関連
[playwright.md](./playwright.md) / [server_client_components.md](./server_client_components.md) / [server_actions.md](./server_actions.md) / [../../react/react19/testing_library.md](../../react/react19/testing_library.md)
