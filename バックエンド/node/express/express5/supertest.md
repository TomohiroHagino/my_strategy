# supertest（Express 5）

## ひとことで言うと
Express の `app` に**直接HTTPリクエストを撃つ**結合テスト用ライブラリ。`request(app).get("/").expect(200)` の形で、実サーバをポートで起動せずにルーティング＋ミドルウェア＋ハンドラを通したレスポンスを検証できる。Jest / Vitest など任意のテストランナーと組み合わせて使う。

## 役割・なぜ必要か
- コントローラを直接呼ぶ単体テストでは、**ルーティング・ミドルウェア・バリデーション・JSONパース**といった「Expressを通る部分」が検証されない。supertest はそこを丸ごと通すので実態に近い。
- `app` を渡すだけで内部的に一時ポートを割り当ててリクエストするため、**ポート競合せず並列実行できる**。`listen` を呼ぶ必要がない。
- ステータス・ヘッダ・JSONボディを宣言的に検証でき、HTTPレベルの仕様書になる。

## 基本の書き方（コード）
```bash
npm i -D supertest
```
```js
// app.js … app と server を分離（テストしやすさの肝）
import express from "express";
const app = express();
app.use(express.json());
app.get("/users/:id", (req, res) => res.json({ id: req.params.id }));
app.post("/users", (req, res) => {
  if (!req.body.name) return res.status(400).json({ error: "name required" });
  res.status(201).json({ id: 1, name: req.body.name });
});
export default app;            // ← app だけ export（listen はしない）
```
```js
// __tests__/users.test.js（Vitest/Jest どちらでも）
import { describe, it, expect } from "vitest";
import request from "supertest";
import app from "../app.js";   // server ではなく app を渡す

describe("GET /users/:id", () => {
  it("200 と id を返す", async () => {
    const res = await request(app).get("/users/42");   // ← await 必須
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ id: "42" });
  });
});

describe("POST /users", () => {
  it("name 無しは 400", async () => {
    const res = await request(app).post("/users").send({});
    expect(res.status).toBe(400);
  });
  it("name ありは 201", async () => {
    const res = await request(app)
      .post("/users")
      .set("Content-Type", "application/json")   // ヘッダ付与
      .send({ name: "太郎" });
    expect(res.status).toBe(201);
    expect(res.body.name).toBe("太郎");
  });
});
```
```js
// .expect() チェーン（supertest 内蔵アサーション）
it("チェーンで検証する", async () => {
  await request(app)
    .get("/users/1")
    .expect("Content-Type", /json/)   // ヘッダの正規表現一致
    .expect(200)                       // ステータス
    .expect((res) => {                 // カスタム検証
      if (!res.body.id) throw new Error("id がない");
    });
});
```
```js
// 認証ヘッダ・クッキーを持ち回る（agent でセッション維持）
const agent = request.agent(app);     // Cookie を保持し続ける
await agent.post("/login").send({ user: "a", pass: "b" });
const res = await agent.get("/dashboard");   // ログイン状態が引き継がれる
expect(res.status).toBe(200);
```

## 実務での使い方・定番パターン
- **`app` を渡す（`server` ではない）**：`app.js` は `export default app`、`listen` は `server.js` に分離。supertest に `app` を渡すと一時ポートで実行され、ポートを占有しないので並列で回せる。これが Express テストの最重要パターン。
- **2つの検証スタイル**：(a) ランナーのマッチャで `expect(res.status).toBe(200)`、(b) supertest 内蔵 `.expect(200)`。前者のほうが失敗メッセージが豊かで、`res.body` を細かく見られるので実務では (a) 寄りが多い。
- **`.send()` / `.set()` / `.query()`**：ボディは `.send({...})`（JSONなら自動で `Content-Type` が付く）、ヘッダは `.set("Authorization", token)`、クエリは `.query({ page: 2 })`。
- **認証フローは `request.agent(app)`**：Cookie/セッションを保持するエージェントを作り、login → 保護リソースの順に叩く。トークン方式なら `.set("Authorization", "Bearer ...")` を都度付ける。
- **テスト用DB＋クリーンアップ**：`NODE_ENV=test` で接続先を切り替え、`beforeEach`/`afterEach` で truncate かトランザクションロールバック。本番DBを叩かないこと（→ [config_env.md](./config_env.md)）。
- **ファイルアップロード**：`.attach("file", "path/to/fixture.png")` でマルチパートを送れる。

## ハマりどころ
- **`await` / `return` 忘れ**：`request(app).get("/")` を await も return もせずに終えると、アサート前にテストが完了して**嘘の緑**になる。`async` テスト＋`await`（または Promise を return）を徹底。
- **`server` を渡してしまう**：`app.listen()` の戻り値（server）を渡すと既にポートを掴んでおり `EADDRINUSE` や後始末漏れに。supertest には**未起動の `app`** を渡す。
- **`Content-Type` 漏れ**：`.send("生文字列")` だと JSON として解釈されない。オブジェクトを `.send({...})` で渡すか `.set("Content-Type", "application/json")` を付ける。
- **`.expect(200)` の取り違え**：supertest の `.expect()` はランナーの `expect()` とは別物。混同して `import { expect }` を上書きしないよう、変数名・import を区別する。
- **エージェントの状態リーク**：`request.agent(app)` は Cookie を保持し続ける。テストごとに新しい agent を作らないとログイン状態が他テストへ漏れる。
- **本番DB/本番環境を叩く**：`NODE_ENV` 切り替えを忘れると本番に書き込む事故。接続先は環境変数で必ず分離。
- **`app/server` 未分離**：`app.js` に `listen` を書いていると import 時点でサーバが起動してしまう。分離が前提。

## 関連
[testing.md](./testing.md) / [vitest.md](./vitest.md) / [request_response.md](./request_response.md) / [config_env.md](./config_env.md)
