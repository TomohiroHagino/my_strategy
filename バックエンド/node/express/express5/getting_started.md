# 始め方（Express 5）

## ひとことで言うと
Node.js の最小限のWebフレームワーク Express を入れて、**HTTPサーバを数行で立ち上げる**ところまでの手順。Express 5 は **Node.js 18 以上**が必須。

## 役割・なぜ必要か
- Node 標準の `http` モジュールだけでもサーバは書けるが、ルーティング・ミドルウェア・リクエスト解析を毎回手書きするのは面倒。Express はそれらを薄く整える。
- 「**まずローカルで `localhost:3000` が動く**」状態を最短で作るのが最初の一歩。ここが土台になり、ルーティングや構成の話に進める。
- Express 5 は依存が更新され（path-to-regexp v8 等）、古い Node を切ったので、まず Node のバージョン確認から始める。

## 基本の書き方（コード）
```bash
# Node のバージョン確認（18以上であること）
node -v   # v18.x 以上ならOK

# プロジェクト初期化（package.json を作る）
npm init -y

# Express 5 を入れる（5系が既定）
npm install express
```

ESM（`import`）で書く場合 — `package.json` に `"type": "module"` を足す:
```js
// app.js （ESM）
import express from "express";

const app = express();

app.get("/", (req, res) => {
  res.send("Hello Express 5");
});

app.listen(3000, () => {
  console.log("listening on http://localhost:3000");
});
```

CommonJS（`require`）で書く場合 — `"type"` は付けない（既定）:
```js
// app.js （CommonJS）
const express = require("express");

const app = express();

app.get("/", (req, res) => {
  res.send("Hello Express 5");
});

app.listen(3000, () => {
  console.log("listening on http://localhost:3000");
});
```

```bash
# 起動
node app.js
# → ブラウザで http://localhost:3000 を開く
```

## 実務での使い方・定番パターン
- **ESM か CommonJS かは最初に決める**。新規なら ESM（`import`）が今後の主流。既存資産や古いライブラリ都合なら CommonJS。途中で混ぜない。
- **保存したら自動再起動**したい。開発中は手で `node app.js` し直すのが面倒なので、ホットリロード系を使う:
```bash
# 素のJS: nodemon（保存で再起動）
npm install --save-dev nodemon

# TypeScript or 最新JSをそのまま: tsx（実行＋watch）
npm install --save-dev tsx
```
- `package.json` の `scripts` にコマンドを登録しておく（チームで揃う）:
```json
{
  "type": "module",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js",
    "dev:tsx": "tsx watch app.js"
  }
}
```
```bash
npm run dev   # 開発時はこれ。保存するたび再起動される
```
- **ポートは環境変数で渡せる**ようにしておくと本番／ローカルで切り替えやすい:
```js
const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`listening on ${port}`));
```

## ハマりどころ / アンチパターン
- **ESM / CommonJS の混同**が最頻トラブル。`import` を使うのに `"type": "module"` を書き忘れると `Cannot use import statement outside a module`。逆に `"type": "module"` を入れたまま `require` を使うと `require is not defined`。**どちらか一方に統一**する。
- ESM では `__dirname` / `__filename` が無い。必要なら `import.meta.url` から作る（`fileURLToPath` 等）。
- **ポート衝突** `EADDRINUSE: address already in use :::3000`。前回のプロセスが残っているか、別アプリが同じポートを使っている。`lsof -i :3000`（mac/Linux）で犯人を探して落とすか、ポートを変える。
- **Node が18未満**だと Express 5 が動かない／警告が出る。`node -v` を最初に確認。バージョン管理は `nvm` / `volta` が定番。
- `app.listen()` を**呼び忘れる**と「コードは動いたのに繋がらない」。リクエストを受けるには listen が必須。
- グローバル `npm install -g express` は不要。**プロジェクトローカルに入れる**（`node_modules`）のが正解。

## フォルダ構成（始動直後）
> **Express も公式の決まった構成は無い**（最小は `app.js` 1枚でも動く）。
> `express-generator` が作るのは `bin/www`・`routes`・`views`・`public` まで。**層（controllers/services/models）は自分で足す**。

```
myapp/
├── app.js                    # アプリ本体・ミドルウェア/ルータ登録（or index.js）
├── bin/
│   └── www                   # 起動スクリプト（http.createServer + listen）
├── routes/                   # ルート定義（パス↔コントローラ）
│   ├── index.js
│   └── users.js              #   例: router.get('/', userController.list)
├── controllers/              # コントローラ（(req,res) で受取・返却）   # 自分で作る
│   └── userController.js
├── services/                 # 業務ロジック                           # 自分で作る
│   └── userService.js
├── models/                   # データモデル/DBアクセス（Prisma/Mongoose/Knex等） # 自分で作る
│   └── userModel.js
├── middleware/               # 認証・エラー処理等の自作MW              # 自分で作る
│   ├── auth.js
│   └── errorHandler.js
├── config/                   # 設定（DB接続・定数）                    # 自分で作る
│   └── db.js
├── views/                    # テンプレート（ejs/pug）【HTMLを返す場合】
│   ├── index.ejs
│   └── error.ejs
├── public/                   # 静的配信（express.static）
│   ├── stylesheets/style.css   javascripts/   images/
├── package.json
├── package-lock.json         # 依存の固定（npm install で生成）
├── .env                      # 環境変数（PORT等）   # 自分で作る
└── .gitignore                # node_modules等の除外  # 自分で作る
```
- **層の発想は Spring と同じ**：`routes`(結びつけ) → **`controllers`((req,res)で受取・返却)** → `services`(業務) → `models`(DB)。これらは規約に無いので自分で足す。
- バリデーションは **express-validator** か **zod** をミドルウェアとして挟む。
- HTML不要のAPIなら `views/` `public/` は不要（`res.json()` 中心）。

## 関連
[project_structure.md](./project_structure.md) / [routing.md](./routing.md)
