# Vitest（Express 5）

## ひとことで言うと
Vite ベースの**テストランナー**。`describe` / `it` / `expect` で書き、`vi.mock` / `vi.fn` でモックする。ESM・TypeScript がそのまま動き起動が速い。API は Jest とほぼ同じで、`jest.fn` → `vi.fn`、`jest.mock` → `vi.mock` の置き換えでほぼ移行できる。Express では HTTP 部分を supertest と組み合わせて使う。

## 役割・なぜ必要か
- Postman で全エンドポイントを手で叩くのは非現実的。**回帰を自動検出**するために要る。
- Vitest は Vite の変換を共有するので **ESM / TS をビルド設定なしで実行**でき、watch も速い。Jest の設定（`ts-jest` / `babel`）で詰まりがちな部分を回避できる。
- `vi.mock` で外部API・メール・DB層を差し替え、決定的（毎回同じ結果）なテストにできる。

## 基本の書き方（コード）
```bash
npm i -D vitest supertest
```
```js
// vitest.config.js（最小。node 環境で十分）
import { defineConfig } from "vitest/config";
export default defineConfig({
  test: { environment: "node", globals: false },
});
```
```js
// __tests__/cart.test.js（純ロジックのユニットテスト）
import { describe, it, expect } from "vitest";
import { calcTotal } from "../lib/cart.js";

describe("calcTotal", () => {
  it("合計を計算する", () => {
    expect(calcTotal([{ price: 100 }, { price: 200 }])).toBe(300);
  });
  it("空配列は0", () => {
    expect(calcTotal([])).toBe(0);
  });
});
```
```js
// vi.fn：コールバック/関数呼び出しの検証
import { it, expect, vi } from "vitest";

it("各要素にコールバックが呼ばれる", () => {
  const spy = vi.fn();
  [1, 2, 3].forEach(spy);
  expect(spy).toHaveBeenCalledTimes(3);
  expect(spy).toHaveBeenCalledWith(2, 1, [1, 2, 3]);
});
```
```js
// vi.mock：モジュールごと差し替え（外部境界をスタブ化）
import { it, expect, vi } from "vitest";
import { sendWelcome } from "../service/user.js";

vi.mock("../lib/mailer.js", () => ({
  sendMail: vi.fn().mockResolvedValue({ id: "msg_1" }),  // 実送信せず即解決
}));

it("登録時にメールを送る", async () => {
  const { sendMail } = await import("../lib/mailer.js");
  await sendWelcome({ email: "a@example.com" });
  expect(sendMail).toHaveBeenCalledOnce();
});
```
```js
// supertest と組み合わせた HTTP 結合テスト（app を渡す）
import { describe, it, expect } from "vitest";
import request from "supertest";
import app from "../app.js";

describe("GET /users/:id", () => {
  it("200 と id を返す", async () => {
    const res = await request(app).get("/users/42");
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ id: "42" });
  });
});
```

## 実務での使い方・定番パターン
- **`globals: false` が既定**：`describe`/`it`/`expect` を毎ファイル `import` する。Jest 風にグローバルで使いたいなら `globals: true` ＋ `vitest/globals` を tsconfig の types に追加。
- **`vi.fn()` / `vi.spyOn()`**：単発の関数モックは `vi.fn()`、既存オブジェクトのメソッドを覗くなら `vi.spyOn(obj, "method")`。`mockResolvedValue` / `mockRejectedValue` で async を制御。
- **`vi.mock()` はホイストされる**：ファイル先頭に巻き上げられるので、モック対象の `import` より前に書いても効く。逆に動的な値を使うときは `vi.hoisted()` で巻き上げる。
- **`beforeEach`/`afterEach` でリセット**：`vi.clearAllMocks()`（呼び出し履歴クリア）/ `vi.resetAllMocks()`（実装もリセット）。`test.clearMocks: true` を config に入れると自動化できる。
- **HTTP は supertest**：Vitest 単体は HTTP を叩かない。ルート結合は `request(app)` で（→ [supertest.md](./supertest.md)）。Express の `app` と `server` を分離し、supertest には `app` を渡す。
- **`environment`**：Express は `node`。DOM が要る場合だけ `jsdom`/`happy-dom`。バックエンドでは `node` のまま軽くする。
- **カバレッジ**：`npm i -D @vitest/coverage-v8` → `vitest run --coverage`（目安80%）。
- **Jest からの移行**：`jest` → `vi`、`jest.fn` → `vi.fn`、`jest.mock` → `vi.mock`。マッチャ（`toBe`/`toEqual`/`toHaveBeenCalled`）はほぼ同じ。supertest はどちらでもそのまま使える。

## ハマりどころ
- **`vi.mock` のパス不一致**：モック対象は**実コードと同じ import パス**で指定する（拡張子・相対の食い違いでモックが効かず実物が走る）。ESM では拡張子まで合わせる。
- **`await` 忘れ**：`request(app).get("/")` を await せず終えると、非同期の失敗が拾われず**嘘の緑**になる。`async` テスト＋`await` を徹底。
- **モック履歴のリーク**：`clearAllMocks` を入れないと前テストの呼び出し回数が残り、`toHaveBeenCalledTimes` が順序依存で落ちる。`beforeEach` でクリア。
- **`globals: true` 前提のコピペ**：他プロジェクトから `import` 無しのテストを持ち込むと「`describe` is not defined」。config の `globals` 設定と揃える。
- **ESM/CJS 混在**：`require` 主体の古いコードに Vitest（ESM 前提）を被せると解決でつまずく。`app.js` 側を ESM に寄せるか `vite` の `deps` 設定で吸収。
- **過剰なモック**：内部実装まで `vi.mock` で縛るとリファクタで赤くなる。モックは外部境界（API・送信・課金）だけに留める。

## 関連
[testing.md](./testing.md) / [supertest.md](./supertest.md) / [project_structure.md](./project_structure.md)
