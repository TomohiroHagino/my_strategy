# Vitest（Hono v4）

## ひとことで言うと
Hono のテストで使う定番**テストランナー**。`describe` / `it` / `expect` で書き、Hono 組込の **`app.request()`** にWeb標準の `Request`（または `init` オブジェクト）を渡して返ってきた `Response` を検証する。サーバを起動せず `fetch` 同等の入出力でルートを叩ける。`vi.fn` / `vi.mock` で外部依存を差し替える。

## 役割・なぜ必要か
- Hono アプリは「`Request` を受けて `Response` を返す関数」そのもの。だから**ランタイム起動不要**で、`app.request('/users')` を呼ぶだけで結合テストができ、Vitest がその実行とアサートを担う。
- Vitest は TypeScript・ESM をビルド設定なしで動かし watch も速い。Workers/Deno/Bun/Node どの環境向けコードでも、テストの書き方は同じになる。
- `vi.fn()` でDBクライアントや外部APIをスタブ化し、`c.env`（バインディング）のモックと組み合わせて決定的なテストにできる。

## 基本の書き方（コード）
```bash
npm i -D vitest
```
```ts
// src/index.ts（テスト対象）
import { Hono } from "hono";
const app = new Hono();
app.get("/users", (c) => c.json([{ id: 1, name: "Taro" }]));
app.post("/users", async (c) => {
  const body = await c.req.json<{ name: string }>();
  return c.json({ id: 2, ...body }, 201);
});
export default app;
```
```ts
// src/index.test.ts（Vitest + app.request）
import { describe, it, expect } from "vitest";
import app from "./index";

describe("users API", () => {
  it("一覧が200を返す", async () => {
    const res = await app.request("/users");      // GET 既定
    expect(res.status).toBe(200);
    const data = await res.json();                 // ← await 必須
    expect(data).toHaveLength(1);
  });

  it("POSTで作成され201を返す", async () => {
    const res = await app.request("/users", {      // 第2引数は fetch の init 同様
      method: "POST",
      body: JSON.stringify({ name: "Hanako" }),
      headers: { "Content-Type": "application/json" },
    });
    expect(res.status).toBe(201);
    expect(await res.json()).toEqual({ id: 2, name: "Hanako" });
  });
});
```
```ts
// c.env（バインディング）が要るルートは第3引数でモックを注入
import { describe, it, expect, vi } from "vitest";
import app from "./index";

it("env依存ルートにモックを渡す", async () => {
  const fakeDb = { get: vi.fn().mockResolvedValue({ id: 1 }) };
  const MOCK_ENV = { DB: fakeDb, TOKEN: "test" };

  const res = await app.request(
    "/protected",
    { headers: { Authorization: "Bearer test" } },
    MOCK_ENV,                                      // ← 第3引数 env
  );
  expect(res.status).toBe(200);
  expect(fakeDb.get).toHaveBeenCalledOnce();
});
```
```ts
// vi.mock：依存モジュールごと差し替え
import { vi } from "vitest";
vi.mock("./lib/mailer", () => ({
  sendMail: vi.fn().mockResolvedValue(true),
}));
```
```bash
npx vitest run            # 一括実行
npx vitest                # watch
npx vitest run --coverage # カバレッジ（@vitest/coverage-v8 が必要）
```

## 実務での使い方・定番パターン
- **`app.request(path, init?, env?)` が主役**：第1引数はパス（または `Request`）、第2引数は `fetch` の `init`（method/body/headers）、**第3引数で `c.env` のモック**を注入。これだけでルート結合が検証できる。
- **AAA構成**：Arrange（入力組み立て）→ Act（`await app.request(...)`）→ Assert（`res.status` と `await res.json()`）。ステータスとボディの両方を見る。
- **`c.env` のモック**：D1・KV・シークレットは第3引数で偽物を渡す。`vi.fn()` でDBクライアントをスタブ化し、外部依存だけ差し替える。env依存ルートを第3引数なしで叩くと `undefined` 参照で落ちる。
- **`vi.fn()` / `vi.mock()`**：単発スタブは `vi.fn()`、モジュール丸ごとは `vi.mock()`（ファイル先頭にホイストされる）。`mockResolvedValue` で async を制御。`beforeEach(() => vi.clearAllMocks())` で履歴をリセット。
- **`vitest.config.ts` の環境**：Node 環境で十分なら素の Vitest。Workers固有API（`caches`・Durable Objects 等）を厳密に再現したいなら `@cloudflare/vitest-pool-workers` を使う。
- **テストピラミッド**：純粋ロジック/ユーティリティはユニット、ルート結合は `app.request`、重要フローだけ実ランタイム上で少数のE2E。
- **RPC（hc クライアント）のテスト**：型安全RPCは `app.request` でサーバ側を検証すれば足りる。クライアント側は型チェックで担保（→ [rpc.md](./rpc.md)）。

## ハマりどころ
- **`res.json()` の `await` 忘れ**：`Response.json()` は **Promise**。`await` を付けないと中身でなく Promise をアサートして落ちる。`res.text()` も同様。
- **`c.env` をモックし忘れる**：env依存ルートを第3引数なしで叩くと `undefined` 参照で落ちる。必要なバインディングを `MOCK_ENV` で渡す。
- **Vitest環境の取り違え**：`caches` や Durable Objects などWorkers固有APIを使うのに Node 環境のまま回すと挙動が合わない。バインディング依存が強いなら workers プールを検討。
- **POSTで `Content-Type` 漏れ**：`c.req.json()` 側がJSONとして読めず失敗。`headers` に `application/json` を必ず付ける。
- **`vi.mock` のパス不一致**：モック対象は実コードと同じ import パスで指定。食い違うとモックが効かず実物が走る。
- **状態リーク**：モジュールスコープの可変状態（キャッシュ等）やモック履歴がテスト間で残る。`beforeEach` で初期化、`vi.clearAllMocks()` を入れる。
- **実サーバを立ててテスト**：`serve()` でポートを開いて叩くのは遅く不安定。`app.request` で十分（起動不要が利点）。

## 関連
[testing.md](./testing.md) / [routing.md](./routing.md) / [rpc.md](./rpc.md)
