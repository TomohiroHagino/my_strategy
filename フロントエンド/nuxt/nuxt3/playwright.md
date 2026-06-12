# Playwright（Nuxt 3）

## ひとことで言うと
**実ブラウザ（Chromium / Firefox / WebKit）を自動操作して、Nuxt アプリを通しで検証する E2E テストツール**。`test` でケースを書き、`page.goto` でページにアクセスし、`locator` で要素を掴み、`expect(locator)` で検証する。SSR で生成された HTML、ハイドレーション後の対話、`server/api` への通信まで含めた **本物の結果** を確認できる。

## 役割・なぜ必要か
- `@nuxt/test-utils` のコンポーネントテストは個別部品の検証。Playwright は **SSR・ハイドレーション・ルーティング・サーバAPI まで通った状態** を実ブラウザで検証する。
- 「初期表示は SSR の HTML、操作後はクライアントで更新」という Nuxt 特有の流れを、ユーザーが見るとおりに確認できる。
- 「ログイン → 一覧 → 詳細」のような **重要ユーザーフロー** を自動で守る。
- 自動待機が組み込まれ、固定 `sleep` なしで安定する。スクリーンショット / トレースで失敗調査がしやすい。

## 基本の書き方（コード）
`test` → `page.goto` → `locator` → `expect`。
```ts
// e2e/home.spec.ts
import { test, expect } from "@playwright/test";

test("トップページがSSRで表示される", async ({ page }) => {
  await page.goto("/");

  await expect(page.getByRole("heading", { name: "Welcome" })).toBeVisible();
});
```
```ts
// playwright.config.ts（本番相当：build → preview を起動して叩く）
import { defineConfig } from "@playwright/test";

export default defineConfig({
  use: { baseURL: "http://localhost:3000" },
  webServer: { command: "npm run build && npm run preview", url: "http://localhost:3000", reuseExistingServer: !process.env.CI },
});
```

## 実務での使い方・定番パターン
- **本番相当ビルドで回す**：`nuxi build`（→ `node .output/server/index.mjs` / `nuxi preview`）を `webServer` にする。SSR・Nitro の挙動は dev と違うため。
- **SSR の初期 HTML を確認**：JS 無効化やレスポンス検証で、ハイドレーション前から見出し・本文が出ているか確かめる（SEO・初回表示の担保）。
- **ロケータはユーザー視点**：`getByRole` / `getByLabel` / `getByText` を優先。`getByTestId` は最後の手段。
- **認証状態の使い回し**：1回ログインして `storageState` を保存し、各テストで読み込む。
- **重要フローだけ**：E2E は遅く壊れやすい。コンポーネントは `@nuxt/test-utils` の `mountSuspended`、結合の最終確認は Playwright、と分担する。
- **CI ではトレース取得**：`trace: "on-first-retry"` で失敗時のみ記録する。

## ハマりどころ
- **dev サーバで検証**：SSR/Nitro が本番と違い、見逃し・誤検知が出る。本番ビルドで回す。
- **ハイドレーション待ち不足**：SSR 直後はまだイベントが付いていない。クリックは要素が操作可能になるまで待つ（`expect(locator)` の自動待機を使う）。
- **固定待ち（`waitForTimeout`）**：サーバ往復をタイマーで待つとフレーク化。`waitForResponse` 等を使う。
- **セレクタが実装依存**：CSS クラスや DOM 構造で掴むとリファクタで壊れる。`getByRole`/`getByText` に寄せる。
- **テスト間の状態共有**：DB や `storageState` が残って順序依存になる。各テストを独立させる。

## 関連
[nuxt_test_utils.md](./nuxt_test_utils.md) / [rendering.md](./rendering.md) / [server_routes.md](./server_routes.md)
