# エラーハンドリング（Error Handling）（Express 5）

## ひとことで言うと
**引数が4個の特別なミドルウェア `(err, req, res, next) => {}`**。Express は「引数が4個」というだけでこれを**エラー専用ハンドラ**と認識し、どこかで `next(err)` が呼ばれたとき（または Express 5 では async ハンドラが reject したとき）ここへジャンプさせる。アプリの「例外の最終受け皿」。

## 役割・なぜ必要か
- 各ハンドラの中で個別に `try/catch` してレスポンスを組み立てると、エラー応答の形式がバラバラになる。**1か所に集約**してステータスとJSONを統一できる。
- 「正常系の流れ」と「異常系の流れ」を分離できる。ハンドラは成功処理に集中し、失敗は `next(err)`（または throw）で投げて後段に任せる。
- ログ出力・通知・機密情報の除去（スタックトレースを本番でクライアントに返さない）も集約できる。

## 基本の書き方（コード）
```js
const express = require("express");
const app = express();
app.use(express.json());

// 任意：独自エラー型（ステータスを持たせる）
class HttpError extends Error {
  constructor(status, message) { super(message); this.status = status; }
}

// 同期 throw → Express 5 は同期 throw も自動でエラーハンドラへ送る
app.get("/sync", (req, res) => {
  throw new HttpError(400, "不正な入力です");
});

// 手動で next(err)：意図的にエラー段へ飛ばす
app.get("/manual", (req, res, next) => {
  const user = null;
  if (!user) return next(new HttpError(404, "見つかりません")); // ← エラーハンドラへ
  res.json(user);
});

// Express 5：async の reject は自動で転送（4では自前ラップが必要だった）
app.get("/async", async (req, res) => {
  const data = await fetchFromDb();   // ここで throw/reject しても
  res.json(data);                     // ↓ 下のエラーハンドラが受け取る
});

// 集約エラーハンドラ：必ず「最後」に、引数「4個」で登録
app.use((err, req, res, next) => {
  const status = err.status || 500;
  console.error(`[${status}] ${err.message}`);            // サーバ側に詳細ログ
  res.status(status).json({ error: err.message });        // クライアントには最小限
});

app.listen(3000);
```

## 実務での使い方・定番パターン
- **エラーハンドラは一番最後に1つ**：全ルート定義の**後ろ**に `app.use((err, req, res, next) => {...})` を置く。ここより前で起きた `next(err)` を拾う。
- **ステータスを持つエラー型**：`err.status` を見て分岐（400/401/404/500…）。`HttpError` のような独自クラスやライブラリ（`http-errors`）が定番。
- **404 ハンドラとエラーハンドラを分ける**：未マッチは「正常終了だが該当なし」なので、ルート群の後・エラーハンドラの前に通常ミドルウェアで返す。
  ```js
  app.use((req, res) => res.status(404).json({ error: "Not Found" })); // 全ルートの後
  app.use((err, req, res, next) => { /* 例外の最終受け皿 */ });        // さらに後
  ```
- **本番ではスタックトレースを出さない**：`message` だけ返し、`err.stack` はサーバログのみ。`process.env.NODE_ENV` で出し分け。→ [config_env.md](./config_env.md)
- **Express 5 の自動転送を活用**：async ハンドラ内は素直に `await` だけ書けばよい。reject は勝手にエラーハンドラへ。→ [async_patterns.md](./async_patterns.md)

## ハマりどころ / アンチパターン
- **引数を4個にしないとエラー用にならない**：最頻出の罠。`(err, req, res) => {}` と3個で書くと、Express は「普通のミドルウェア」と判定し `err` を渡してくれない。**必ず `next` まで4個**書く（使わなくても）。
- **Express 4 と 5 の差を混同**
  - **4**：async ハンドラの reject は**自動転送されない**。`try/catch` で `next(err)` するか `express-async-handler` 等でラップが必要だった。
  - **5**：async の reject も**同期 throw も自動でエラーハンドラへ転送**。4向けの手動ラップは不要（残っていても害は少ないが冗長）。
- **コールバック内の throw は捕まらない**：`setTimeout` や非async のコールバック中の throw は Express の自動転送対象外。その場合は `try/catch` して `next(err)`。
- **エラーハンドラを前に置く**：ルート定義より前に書くと、後で起きたエラーを拾えない。**順序が全て**（→ [middleware.md](./middleware.md)）。
- **エラーハンドラ内で再び throw / 二重応答**：ここで例外を出すと Express 既定ハンドラに落ちる。応答は1回、`return` で抜ける。
- **機密漏えい**：`res.json(err)` でエラーオブジェクト丸ごと返すとスタックや内部情報が漏れる。返すのは `message` 等の最小限に。

## 関連
[middleware.md](./middleware.md) / [async_patterns.md](./async_patterns.md) / [request_response.md](./request_response.md)
