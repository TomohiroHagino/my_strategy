# ルーティング（Express 5）

## ひとことで言うと
「**どのURL・どのHTTPメソッドに、どの処理を割り当てるか**」の対応づけ。`app.get("/users", handler)` のように書く、Express の入口。

## 役割・なぜ必要か
- HTTPリクエストは「メソッド（GET/POST/…）＋パス（/users/3）」で来る。これを受けて「この処理を実行する」と結びつけるのがルーティング。
- Express の本質は「**ミドルウェアの列を順に通す**」こと。ルートも実体はミドルウェアで、**上から順にマッチを試す**。だから定義の順番が結果を変える。
- 規模が大きくなると `app` に全ルートを書くと破綻するので、`express.Router()` で機能ごとに分割してマウントする。

## 基本の書き方（コード）
```js
import express from "express";
const app = express();

app.get("/users", (req, res) => res.json([]));          // 一覧
app.post("/users", (req, res) => res.status(201).end()); // 作成
app.put("/users/:id", (req, res) => res.end());          // 更新
app.delete("/users/:id", (req, res) => res.end());       // 削除（app.del は廃止）

// ルートパラメータ → req.params で取れる
app.get("/users/:id", (req, res) => {
  res.json({ id: req.params.id }); // /users/3 → { id: "3" }（文字列）
});

app.listen(3000);
```

`express.Router()` で分割してマウント:
```js
// routes/users.js
import { Router } from "express";
const router = Router();
router.get("/", (req, res) => res.json([]));     // 実体は GET /users
router.get("/:id", (req, res) => res.json({}));  // 実体は GET /users/:id
export default router;

// app.js
import usersRouter from "./routes/users.js";
app.use("/users", usersRouter); // ← "/users" を前置してマウント
```

## 実務での使い方・定番パターン
- **マッチ順を意識する**。具体的なパスを先に、可変パスを後に置く。逆だと飲み込まれる:
```js
app.get("/users/me", meHandler);   // 先：固定パス
app.get("/users/:id", showHandler); // 後：可変。順が逆だと "me" が :id に吸われる
```
- **Express 5 のルート記法（path-to-regexp v8）**。4から移行する時に最も引っかかる所:
```js
// ワイルドカードは「名前付き」必須。* 単体は不可
app.get("/files/*splat", (req, res) => {
  res.json(req.params.splat); // マッチした残り（配列）が splat に入る
});

// 任意パラメータ：旧 "/:id?" は廃止 → 中括弧で囲む
app.get("/users{/:id}", (req, res) => {
  res.json({ id: req.params.id }); // /users でも /users/3 でもマッチ
});

// ルート文字列内の「生の正規表現」は制限された
// 旧: app.get(/\/abc.*/) のような書き方は不可寄りに。
// 必要なら名前付きパラメータ＋ハンドラ内で検証する形へ寄せる
```
- **廃止APIに注意（4→5）**：`app.del()` は削除 → `app.delete()` を使う。`res.json(obj, status)` は不可 → `res.status(x).json(obj)`。`req.param(name)` も廃止 → `req.params` / `req.query` / `req.body` を直接見る。
- **ルート単位でミドルウェアを差せる**（認証など）。ハンドラの前に並べる:
```js
app.get("/admin", requireAuth, adminHandler); // requireAuth が next() で通せば adminHandler へ
```

## ハマりどころ / アンチパターン
- **ルート定義順のミス**：可変パス（`/:id`）を固定パス（`/me`）より上に書くと固定側に届かない。**具体 → 一般**の順に並べる。
- **Express 5 の記法変更を見落とす**：`*` 単体・`/:id?`・生正規表現を4の感覚で書くと `TypeError: Missing parameter name` 等で起動すら失敗する。ワイルドカードは `/*name`、任意は `{/:id}`。
- **`req.params` は常に文字列**。`/users/3` の `id` は `"3"`。数値比較やDB検索の前に `Number(req.params.id)` 等で変換し、不正値も検証する。
- **`app.del()` をそのまま使う**：5では存在しない。`app.delete()` へ。
- **Router のマウント先と内部パスの二重前置**：`app.use("/users", router)` した上で router 内で `router.get("/users")` と書くと `/users/users` になる。Router 内は前置分を除いた相対パスで書く。
- **末尾スラッシュやメソッド違い**で「404になる」勘違い。`GET /users/` と `GET /users`、`POST` と `GET` は別物。叩いているメソッド・パスを確認する。

## 関連
[middleware.md](./middleware.md) / [request_response.md](./request_response.md)
