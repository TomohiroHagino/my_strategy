# Playwright（React）

## ひとことで言うと
**実ブラウザ（Chromium / Firefox / WebKit）を自動操作して、アプリ全体を通しで検証する E2E テストツール**。`test` でケースを書き、`page.goto` でアクセスし、`locator` で要素を掴み、`expect(locator)` で表示・状態を検証する。ビルドして起動した本物のアプリを操作するので、コンポーネント単体テストでは見えない結合不具合を捕まえる。

## 役割・なぜ必要か
- RTL のコンポーネントテストは jsdom 上の単体検証。Playwright は **実ブラウザで API・ルーティング・描画まで通った状態** を検証する。
- 「ログイン → 一覧 → 詳細 → 購入」のような **重要ユーザーフロー** を自動で守る。手動リグレッションの置き換え。
- 自動待機（要素が出る・操作可能になるまで待つ）が組み込まれており、固定 `sleep` なしで安定する。
- スクリーンショット / トレース / 動画で失敗時の原因調査がしやすい。

## 基本の書き方（コード）
`test`（ケース）→ `page.goto`（アクセス）→ `locator`（探す）→ `expect`（検証）。
```ts
// e2e/counter.spec.ts
import { test, expect } from "@playwright/test";

test("ボタンを押すとカウントが増える", async ({ page }) => {
  await page.goto("/");

  await page.getByRole("button", { name: "増やす" }).click();

  await expect(page.getByText("カウント: 1")).toBeVisible();
});
```
```ts
// playwright.config.ts（dev サーバを自動起動して叩く）
import { defineConfig } from "@playwright/test";

export default defineConfig({
  use: { baseURL: "http://localhost:5173" },
  webServer: { command: "npm run dev", url: "http://localhost:5173", reuseExistingServer: true },
});
```

## 実務での使い方・定番パターン
- **ロケータはユーザー視点**：`getByRole` / `getByLabel` / `getByText` を優先。`getByTestId` は最後の手段。RTL と思想が揃う。
- **`expect(locator)` の自動リトライ**：`toBeVisible` / `toHaveText` 等は条件が満たされるまで待つ。`await expect(...)` を必ず付ける。
```ts
await expect(page.getByRole("heading")).toHaveText("ダッシュボード");
```
- **Page Object パターン**：ページごとの操作（ログイン手順など）をクラスにまとめて使い回す（DRY・壊れにくい）。
- **認証状態の使い回し**：1回ログインして `storageState` を保存し、各テストで読み込む。毎回ログインしない。
- **重要フローだけ**：E2E は遅く壊れやすい。テストピラミッドの頂点として **少数の主要フロー** に絞る。土台は RTL のユニット/コンポーネントテスト。
- **CI ではトレース取得**：`trace: "on-first-retry"` で失敗時のみ記録し、原因を後追いできる。

## ハマりどころ
- **固定待ち（`waitForTimeout`）**：環境差でフレーク化する。`expect(locator)` の自動待機やイベント待ち（`waitForResponse` 等）を使う。
- **セレクタが実装依存**：CSS クラスや DOM 構造で掴むとリファクタで壊れる。`getByRole`/`getByText` に寄せる。
- **テスト間の状態共有**：DB やログイン状態が残って順序依存になる。各テストを独立させる（フィクスチャ・状態リセット）。
- **dev ビルド前提のズレ**：本番ビルドと挙動が違うことがある。重要フローは本番相当ビルドでも回す。
- **並列実行の競合**：同じユーザー・同じデータを複数テストが触ると壊れる。テストごとにデータを分ける。

## 関連
[testing_library.md](./testing_library.md) / [testing.md](./testing.md)（全体方針・ランナー） / [msw.md](./msw.md)
