# テスト（Testing）（Express 5）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。定番は **Jest**（または **Vitest**）＋ **supertest**。supertest が Express の `app` に直接 HTTP リクエストを撃ち、レスポンスを検証する。

## 役割・なぜ必要か
- 変更のたびに Postman で全エンドポイントを手で叩くのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- supertest を使うと **実際にサーバをポートで起動しなくても**（`app` を渡すだけで）HTTP レベルの結合テストが書ける。速くて並列に回せる。
- 「この仕様で動く」という実行可能な仕様書になり、リファクタの安全網になる。

## 基本の書き方（コード）
```js
// app.js … app と server を分離（テストしやすさの肝）
const express = require("express");
const app = express();
app.use(express.json());
app.get("/users/:id", (req, res) => res.json({ id: req.params.id }));
app.post("/users", (req, res) => {
  if (!req.body.name) return res.status(400).json({ error: "name required" });
  res.status(201).json({ id: 1, name: req.body.name });
});
module.exports = app; // ← app だけ export（listen はしない）
```
```js
// server.js … 起動はここだけ。本番/開発のエントリ
const app = require("./app");
app.listen(process.env.PORT || 3000);
```
```js
// __tests__/users.test.js（Jest ＋ supertest：HTTP結合テスト）
const request = require("supertest");
const app = require("../app"); // server ではなく app を渡す

describe("GET /users/:id", () => {
  test("200 と id を返す", async () => {
    // Arrange / Act
    const res = await request(app).get("/users/42");
    // Assert
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ id: "42" });
  });
});

describe("POST /users", () => {
  test("name 無しは 400", async () => {
    const res = await request(app).post("/users").send({});
    expect(res.status).toBe(400);
  });
  test("name ありは 201", async () => {
    const res = await request(app).post("/users").send({ name: "太郎" });
    expect(res.status).toBe(201);
    expect(res.body.name).toBe("太郎");
  });
});
```
```js
// ユニットテスト：純粋な関数を単体で検証（HTTP を介さない）
const { calcTotal } = require("../lib/cart");
test("合計を計算する", () => {
  expect(calcTotal([{ price: 100 }, { price: 200 }])).toBe(300);
});
```

## 実務での使い方・定番パターン
- **テストピラミッド**：土台に多数の高速な **ユニットテスト**（純粋関数・ロジック）、中間に **結合テスト**（supertest でルート＋ミドルウェア＋バリデーションを通す）、頂点に少数の **E2E**（Playwright 等で実サーバ）。
- **`app` と `server` を分離**：`app.js` は `module.exports = app`、`listen` は `server.js` に置く。supertest には **`app` を渡す**（ポートを占有せず並列実行できる）。これが Express テストの最重要パターン。
- **モック**：外部 API・メール送信・決済などはモックして決定的に。Jest なら `jest.mock("../lib/mailer")`、Vitest なら `vi.mock(...)`。DB アクセス層も境界でモックすると速い。
- **テスト用 DB / 環境**：`NODE_ENV=test` で接続先を切り替え、テスト用 DB（本番と別）を使う。各テストの前後で `beforeEach`/`afterEach` でデータをクリーン。実 DB を使う結合テストは Docker / インメモリ（sqlite, mongodb-memory-server）が定番。
- **非同期は必ず `async/await`**：`await request(app)...` で待つ。await を忘れるとアサート前にテストが終わって誤って緑になる。
- **Vitest を選ぶ場面**：ESM / Vite 構成や高速起動が欲しいとき。API は Jest とほぼ同じ（`describe`/`test`/`expect`）。supertest はそのまま使える。

## ハマりどころ / アンチパターン
- **app/server を分離していない**：`app.listen()` を `app.js` に書くと、テスト中にポートを掴んでしまい `EADDRINUSE` や後始末漏れに。`app` だけ export して supertest に渡す形へ。
- **`await` 忘れ**：`request(app).get("/")` を await せず終わると、非同期の失敗が拾われず**テストが嘘の緑**になる。`async` テスト＋`await` を徹底。
- **テスト間の状態リーク**：モジュールスコープのキャッシュ・グローバル変数・共有 DB が残り、順序依存で落ちる。`beforeEach` で初期化、テスト用 DB はトランザクション or truncate。
- **本番 DB / 本番環境を叩く**：`NODE_ENV` 切り替えを忘れて本番に書き込む事故。接続先は環境変数で必ず分離。→ [config_env.md](./config_env.md)
- **過剰なモック**：内部実装まで縛ると、リファクタで赤くなる「もろい」テストに。モックは外部境界（API・送信・課金）だけに留める。
- **`app.listen` の戻り（server）を閉じ忘れ**：実サーバを起動する E2E では `afterAll(() => server.close())` を忘れるとプロセスが終わらない。

## 関連
[vitest.md](./vitest.md) / [supertest.md](./supertest.md) / [project_structure.md](./project_structure.md) / [request_response.md](./request_response.md) / [config_env.md](./config_env.md)
