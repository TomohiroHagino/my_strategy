# MSW（Mock Service Worker）（React）

## ひとことで言うと
**ネットワーク層で API リクエストを横取りして、用意したダミー応答を返すモックライブラリ**。`http.get` などでハンドラを定義し、`setupServer`（テスト）/ `setupWorker`（ブラウザ）で起動する。アプリ側のコード（`fetch` / axios）は一切変えずに、実際の HTTP 通信だけを差し替える。

## 役割・なぜ必要か
- `fetch` を直接モックすると、呼び出し方を変えただけでテストが壊れる。MSW は **HTTP という境界** で差し替えるので、実装に近く壊れにくい。
- テスト・開発・E2E で **同じハンドラを使い回せる**（バックエンド未完成でもフロントを進められる）。
- 「200 で正常」「500 でエラー」「遅延でローディング」など、**異常系・境界条件を自在に再現** できる。
- アプリのコードに分岐やモック差し込みを書かずに済む（本番コードを汚さない）。

## 基本の書き方（コード）
ハンドラ定義 → `setupServer` で起動 → テストから使う。
```ts
// mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users/:id", ({ params }) => {
    return HttpResponse.json({ id: params.id, name: "Taro" });
  }),
];
```
```ts
// mocks/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```
```ts
// vitest.setup.ts（テスト前後でライフサイクルを管理）
import { server } from "./mocks/server";

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers()); // テストごとに上書きを破棄
afterAll(() => server.close());
```
```tsx
// UserCard.test.tsx（アプリのfetchはそのまま、応答だけモックされる）
test("取得したユーザー名が表示される", async () => {
  render(<UserCard id="1" />);
  expect(await screen.findByText("Taro")).toBeInTheDocument();
});
```

## 実務での使い方・定番パターン
- **デフォルトは正常系、テスト内で上書き**：個別ケースだけ `server.use(...)` でエラー応答に差し替える。
```ts
test("APIが500ならエラー表示", async () => {
  server.use(
    http.get("/api/users/:id", () => new HttpResponse(null, { status: 500 })),
  );
  render(<UserCard id="1" />);
  expect(await screen.findByText("読み込みに失敗しました")).toBeInTheDocument();
});
```
- **`onUnhandledRequest: "error"`**：ハンドラ未定義の通信を即エラーにし、モック漏れに気づける。
- **開発時はブラウザで**：`setupWorker` ＋ Service Worker で、ブラウザ実行中も同じハンドラでモックできる。
- **TanStack Query / SWR と相性良し**：データ取得ライブラリの裏側で透過的に効く。→ [data_fetching.md](./data_fetching.md)

## ハマりどころ
- **`resetHandlers()` 忘れ**：`server.use` の上書きが次のテストに漏れて順序依存で落ちる。`afterEach` で必ずリセット。
- **パスのズレ**：相対パス（`/api/...`）と絶対 URL（`https://...`）の不一致でハンドラが当たらない。アプリの実呼び出しと揃える。
- **Service Worker ファイル未生成**：ブラウザ利用時は `npx msw init public/` が必要。忘れると起動しない。
- **テスト環境での未対応 API**：古い Node では `fetch` ポリフィルが要ることがある。ランナー設定を確認する。
- **過剰モック**：内部関数までモックすると壊れやすい。MSW は **HTTP 境界だけ** に使う。

## 関連
[testing_library.md](./testing_library.md) / [testing.md](./testing.md)（全体方針） / [data_fetching.md](./data_fetching.md) / [playwright.md](./playwright.md)
