# リクエスト / レスポンス（req / res）（Express 5）

## ひとことで言うと
ハンドラ `(req, res) => {}` に渡る2つのオブジェクト。**`req`＝クライアントから来た情報の入れ物**（URL・クエリ・ボディ・ヘッダ）、**`res`＝クライアントに返すための道具**（ステータス・JSON・リダイレクト）。この2つを読み書きするのがハンドラの仕事。

## 役割・なぜ必要か
- `req` から「何を求められているか」を読み取り、`res` で「どう応答するか」を決める。HTTPの入出力をオブジェクトの形で扱えるようにした抽象。
- `req` の各プロパティは**出どころが違う**：URLの一部（`params`）／URL末尾の検索条件（`query`）／本文（`body`）／ヘッダ（`headers`）。混同するとデータが取れない。
- `res` は**1リクエストにつき1回だけ**応答を返す（送り終えたら再送不可）。

## 基本の書き方（コード）
```js
const express = require("express");
const app = express();
app.use(express.json());   // ← これが無いと req.body は undefined

// GET /users/42?sort=desc
app.get("/users/:id", (req, res) => {
  const id   = req.params.id;        // "42"（URLパスの :id 部分）
  const sort = req.query.sort;       // "desc"（?以降のクエリ）
  const ua   = req.headers["user-agent"]; // リクエストヘッダ
  res.json({ id, sort, ua });        // JSON で返す（Content-Type 自動付与）
});

// POST /users  body: { "name": "alice" }
app.post("/users", (req, res) => {
  const name = req.body.name;        // "alice"（JSON ボディ。json() 必須）
  if (!name) {
    return res.status(400).json({ error: "name は必須です" }); // ステータス＋JSON
  }
  res.status(201).json({ id: 1, name }); // 201 Created
});

// テキスト／リダイレクト／ヘッダ操作
app.get("/hello", (req, res) => res.send("こんにちは"));     // 文字列をそのまま
app.get("/old",   (req, res) => res.redirect("/new"));       // 302 リダイレクト
app.get("/raw",   (req, res) => {
  res.set("X-Custom", "1");          // レスポンスヘッダ追加
  res.status(200).send("ok");
});

app.listen(3000);
```

## 実務での使い方・定番パターン
- **`req` の使い分け**
  - `req.params` … ルートの `:id` 等。**文字列**で来るので数値は `Number()` 変換。
  - `req.query` … `?page=2&limit=10`。全て文字列。ページネーションや検索条件に。
  - `req.body` … POST/PUT の本文。**`express.json()` か `express.urlencoded()` が前提**。→ [middleware.md](./middleware.md)
  - `req.headers` … 認証トークン（`authorization`）・Content-Type 等。キーは小文字。
- **`res` の定番**
  - `res.json(obj)` … API応答の基本。オブジェクトをJSON化。
  - `res.send(...)` … 文字列・Buffer・オブジェクトを型に応じて送る。
  - `res.status(code)` … チェーンして `res.status(404).json(...)` が定石。
  - `res.redirect(url)` … リダイレクト（既定302、`res.redirect(301, url)` も可）。
  - `res.set(name, value)` … ヘッダ付与。`res.sendFile(path)` … ファイル送信。
- **必ず1回だけ応答**：分岐したら `return res....` で抜ける癖をつける（二重送信防止）。

## ハマりどころ / アンチパターン
- **`req.body` が `undefined`**：最頻出。`express.json()`（または `urlencoded`）を**ルートより前**に `app.use` し忘れている。または送信側の `Content-Type` がJSONになっていない。→ [middleware.md](./middleware.md)
- **Express 5 のシグネチャ変更（4から移行で要注意）**
  - **`res.json(obj, status)` は廃止** → `res.status(x).json(obj)` に書き換える。
  - **`res.send(status)`（数値1引数）廃止** → `res.sendStatus(status)` を使う。
  - **`res.sendfile` → `res.sendFile`**（大文字F）に変更。
  - **`req.param(name)` 廃止** → `req.params.name` / `req.query.name` / `req.body.name` を直接使う。
- **`params`/`query` を数値扱いしてバグ**：全て文字列。`req.query.page > 10` のような比較は文字列比較になる。`Number()` してから比較。
- **二重応答**：`Cannot set headers after they are sent` は「すでに応答済みなのに再び `res` を触った」サイン。分岐後の `return` 忘れが原因。
- **ヘッダは応答送信前に**：`res.send()` の後で `res.set()` しても効かない。ヘッダ系は本文送信より前に。

## 関連
[middleware.md](./middleware.md) / [routing.md](./routing.md) / [error_handling.md](./error_handling.md)
