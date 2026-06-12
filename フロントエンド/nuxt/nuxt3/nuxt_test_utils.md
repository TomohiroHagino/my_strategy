# @nuxt/test-utils（Nuxt 3）

## ひとことで言うと
**Nuxt の実行環境込みでコンポーネントやエンドポイントをテストする公式ユーティリティ**。`mountSuspended` で Nuxt の自動インポート・`useFetch`・非同期セットアップを効かせたままコンポーネントをマウントし、`setup` でテストファイルごとに本物の Nuxt サーバを起動して E2E ヘルパー（`$fetch` / `createPage`）を使う。素の Vue Test Utils では動かない「Nuxt 依存」を解決する。

## 役割・なぜ必要か
- Nuxt のコンポーネントは `useFetch` / `useState` / 自動インポートなど **Nuxt ランタイム前提** で書かれる。Vue Test Utils 単体ではこれらが未定義で落ちる。
- `mountSuspended` は **Nuxt 環境を立てた上でマウント** するので、自動インポートや非同期 `setup`（Suspense）がそのまま動く。
- `setup` ＋ `$fetch` / `createPage` で、**実際に起動した Nuxt アプリ**に対する結合・E2E テストが書ける。
- ランナー（Vitest）と `@nuxt/test-utils/runtime` を組み合わせ、テストを Nuxt 環境で実行する。

## 基本の書き方（コード）
コンポーネント（runtime）テストは `mountSuspended`。
```ts
// vitest.config.ts
import { defineVitestConfig } from "@nuxt/test-utils/config";

export default defineVitestConfig({
  test: { environment: "nuxt" }, // Nuxt環境でテストを走らせる
});
```
```ts
// AppCounter.nuxt.spec.ts（mountSuspended：自動インポート/非同期setup込み）
import { mountSuspended } from "@nuxt/test-utils/runtime";
import { it, expect } from "vitest";
import AppCounter from "~/components/AppCounter.vue";

it("ボタンクリックで表示が増える", async () => {
  const wrapper = await mountSuspended(AppCounter); // await 必須（Suspense解決を待つ）
  await wrapper.find("button.increment").trigger("click");
  expect(wrapper.find(".count").text()).toBe("1");
});
```
```ts
// app.e2e.spec.ts（setup：本物のNuxtを起動して $fetch / ページ操作）
import { setup, $fetch, createPage } from "@nuxt/test-utils/e2e";
import { describe, it, expect } from "vitest";

describe("App E2E", async () => {
  await setup({ server: true }); // テストごとにNuxtサーバ起動

  it("server/api が JSON を返す", async () => {
    const data = await $fetch("/api/health");
    expect(data).toEqual({ ok: true });
  });

  it("トップページに見出しが出る", async () => {
    const page = await createPage("/");
    expect(await page.getByRole("heading").textContent()).toContain("Welcome");
  });
});
```

## 実務での使い方・定番パターン
- **`mountSuspended` を既定に**：Nuxt コンポーネントは `useFetch`・自動インポート前提なので、素の `mount` より `mountSuspended` を使う。**必ず `await`**（非同期 `setup` の解決を待つ）。
- **`mockNuxtImport` で合成関数を差し替え**：`useFetch` / `useRoute` などをテスト内でモックし、API 応答やルートを固定する。
- **`registerEndpoint` で API をモック**：`server/api` 相当の応答を登録し、コンポーネントの取得を安定させる。
- **`setup` + `$fetch`**：`server/api` のエンドポイントを直接叩いて入出力を検証（結合テスト）。
- **`createPage`**：Nuxt 起動済みアプリを Playwright ベースで操作（ナビゲーション・SSR 結果の確認）。本格 E2E は Playwright 単体へ。→ [playwright.md](./playwright.md)
- **composables は単体テスト**：UI を介さず関数として検証。Nuxt 依存があれば `environment: "nuxt"` 配下で。

## ハマりどころ
- **`await` 忘れ**：`mountSuspended` / `setup` / `createPage` は非同期。await しないと未解決の状態を検証して落ちる。
- **環境指定漏れ**：`environment: "nuxt"` か `.nuxt.spec.ts` 命名にしないと自動インポートが効かず未定義エラー。
- **`setup` は重い**：本物の Nuxt を起動するため遅い。E2E は少数の主要フローに絞り、コンポーネントテストは `mountSuspended` で速く回す。
- **モック漏れ**：`useFetch` 未モックで実通信が走り不安定。`mockNuxtImport` / `registerEndpoint` で境界を固定する。
- **DOM 更新待ち忘れ**：Vue の再描画は非同期。`trigger` の直後は `await` か `nextTick` を挟む。

## 関連
[playwright.md](./playwright.md) / [data_fetching.md](./data_fetching.md) / [server_routes.md](./server_routes.md) / [../../vue/vue3/vue_test_utils.md](../../vue/vue3/vue_test_utils.md)
