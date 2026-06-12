# プロジェクト構成（Express 5）

## ひとことで言うと
Express は **無規約（unopinionated）**＝「ここにこう置け」という決まりが無いフレームワーク。だからこそ**自分（チーム）で構成を決める**必要がある、その定番の整え方。

## 役割・なぜ必要か
- Rails のような「規約に従えば自動で整う」仕組みが Express には無い。`app.js` 一枚に全部書いても動く。**動くがゆえに放置すると肥大化する**。
- 最初に層（レイヤ）を切っておくと、責務が分かれてテスト・変更がしやすい。逆に決めないとチームごとに置き場がバラバラになり、レビューも保守も辛くなる。
- 特に「**アプリの定義**」と「**サーバの起動**」を分けておくと、テストが圧倒的に書きやすくなる（後述）。

## 基本の書き方（コード）
定番のレイヤリング（小〜中規模の出発点）:
```
src/
├── app.js            # express() の定義・ミドルウェア・ルートのマウント（listenしない）
├── server.js         # app を import して listen するだけ
├── routes/           # URLとcontrollerの対応づけ（express.Router）
│   └── users.js
├── controllers/      # req/res を受け、servicesを呼んで返す（HTTPの担当）
│   └── usersController.js
├── services/         # 業務ロジック（HTTPを知らない。再利用・テスト容易）
│   └── usersService.js
└── models/           # DBアクセス（Prisma/Sequelize/Mongoose等）
    └── user.js
```

`app.js`（アプリ定義。listen しない）:
```js
// src/app.js
import express from "express";
import usersRouter from "./routes/users.js";

const app = express();
app.use(express.json());        // ボディ解析（Express内蔵）
app.use("/users", usersRouter); // /users 以下を Router にマウント

export default app;             // 起動はしない。appを公開するだけ
```

`server.js`（起動だけ）:
```js
// src/server.js
import app from "./app.js";

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`listening on ${port}`));
```

## 実務での使い方・定番パターン
- **`express.Router()` で機能ごとに分割**してマウントする。これが Express の「分割統治」の基本道具:
```js
// src/routes/users.js
import { Router } from "express";
import * as ctrl from "../controllers/usersController.js";

const router = Router();
router.get("/", ctrl.list);       // GET /users
router.get("/:id", ctrl.show);    // GET /users/:id
router.post("/", ctrl.create);    // POST /users
export default router;
```
- **controller は薄く、service に業務ロジックを寄せる**（Rails の Fat Model 思想に近い）。controller は「req から値を取り、service を呼び、res で返す」だけ:
```js
// src/controllers/usersController.js
import * as service from "../services/usersService.js";

export async function show(req, res) {
  const user = await service.findById(req.params.id);
  if (!user) return res.status(404).json({ error: "not found" });
  res.json(user);
}
```
- **`app.js` と `server.js` 分離の効能**＝テストで `app` を import して supertest に渡せる。実際にポートを listen せずにHTTPを叩ける → [testing.md](./testing.md)
- **層は機能で増やす**。`middlewares/`（認証・ロギング）、`config/`（設定）、`lib/`（共通ユーティリティ）を必要になった時に足す。最初から全部作らない（YAGNI）。

## ハマりどころ / アンチパターン
- **巨大 `app.js`**：ルートもロジックもDBも全部 `app.js` に書く → 数百行で手がつけられなくなる。最初に Router で割る。
- **層をまたいで責務が漏れる**：controller に SQL を直書き、service が `res` を触る、など。`res.json()` は controller まで、業務ロジックは service まで、と線を引く。
- **無秩序な肥大**：「とりあえず動くから」で同じディレクトリに増やし続け、`utils.js` が何でも置き場になる。意味のある単位で分ける。
- **構成がチームでばらつく**：規約が無いので人によって置き場が違う → README か `CONTRIBUTING` に**構成ルールを明文化**しておく。
- **`app.listen()` を `app.js` に書いてしまう**：テストや別エントリで import した瞬間に勝手にポートを掴む。listen は `server.js` だけに。
- 過剰な抽象化（使う予定の無い DI コンテナ等）も逆効果。**規模に見合う最小の層**から始める。

## 関連
[routing.md](./routing.md) / [testing.md](./testing.md)
