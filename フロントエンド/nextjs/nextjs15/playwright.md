# Playwright（Next.js 15）

## ひとことで言うと
**実ブラウザ（Chromium / Firefox / WebKit）を自動操作して、App Router のページを通しで検証する E2E テストツール**。`test` でケースを書き、`page.goto` でルートにアクセスし、`locator` で要素を掴み、`expect(locator)` で検証する。Server Component の描画・SSR/RSC・Server Actions まで通った **本物の結果** を確認できるのが、RTL では届かない領域。

## 役割・なぜ必要か
- App Router の Server Component（async）は RTL では描画できない。**サーバで取得して描画されたページ** を実ブラウザで検証できるのは E2E だけ。
- ルーティング（動的ルート・レイアウト・loading/error）まで含めた **ユーザーフロー** を守る。
- Server Actions のフォーム送信 → サーバ処理 → 再描画という流れを、本物の経路で確認できる。
- 自動待機が組み込まれ、固定 `sleep` なしで安定する。スクリーンショット / トレースで失敗調査がしやすい。

## 基本の書き方（コード）
`test` → `page.goto`（App Router のルート）→ `locator` → `expect`。
```ts
// e2e/users.spec.ts
import { test, expect } from "@playwright/test";

test("ユーザー詳細ページがSSRで表示される", async ({ page }) => {
  await page.goto("/users/1"); // app/users/[id]/page.tsx（Server Component）

  await expect(page.getByRole("heading", { name: "Taro さんのページ" })).toBeVisible();
});
```
```ts
// playwright.config.ts（next start を起動して本番相当を叩く）
import { defineConfig } from "@playwright/test";

export default defineConfig({
  use: { baseURL: "http://localhost:3000" },
  webServer: { command: "npm run build && npm run start", url: "http://localhost:3000", reuseExistingServer: !process.env.CI },
});
```

## 実務での使い方・定番パターン
- **本番相当ビルドで回す**：`next build && next start` を `webServer` にする。RSC・キャッシュ・最適化は dev と挙動が違うため。
- **Server Actions のフォーム**：入力 → 送信ボタンクリック → 再描画後の表示を `expect(locator)` で待つ。サーバ往復が入るので自動待機が効く。
```ts
test("フォーム送信で一覧に追加される", async ({ page }) => {
  await page.goto("/todos");
  await page.getByLabel("内容").fill("買い物");
  await page.getByRole("button", { name: "追加" }).click();
  await expect(page.getByText("買い物")).toBeVisible();
});
```
- **動的ルート / レイアウト**：`/users/[id]` などパラメータ違いを複数ケースで叩き、loading・error 境界も確認する。
- **認証状態の使い回し**：1回ログインして `storageState` を保存し、各テストで読み込む。
- **重要フローだけ**：E2E は遅く壊れやすい。Client Component の単体は RTL、Server Component の結果は E2E、と役割分担する。

## ハマりどころ
- **dev サーバで検証**：RSC キャッシュや最適化が本番と違い、見逃し・誤検知が出る。本番ビルドで回す。
- **固定待ち（`waitForTimeout`）**：Server Action のサーバ往復をタイマーで待つとフレーク化。`expect(locator)` の自動待機や `waitForResponse` を使う。
- **セレクタが実装依存**：CSS クラスや DOM 構造で掴むとリファクタで壊れる。`getByRole`/`getByText` に寄せる。
- **テスト間の状態共有**：DB や `storageState` が残って順序依存になる。データを分け、各テストを独立させる。
- **環境変数の不足**：ビルド時に必要な env が無いと `next build` で落ちる。CI に設定する。

## 関連
[testing_library.md](./testing_library.md) / [server_client_components.md](./server_client_components.md) / [server_actions.md](./server_actions.md) / [routing.md](./routing.md)
