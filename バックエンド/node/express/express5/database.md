# DB（Prisma / Sequelize / Mongoose、内蔵なし）（Express 5）

## ひとことで言うと
**Express にはデータ層（ORM・DB ドライバ）が無い**。Rails の Active Record のような標準は存在せず、**DB もライブラリも自分で選ぶ**。SQL 系なら **Prisma** / Sequelize / TypeORM、MongoDB なら **Mongoose** を別途入れて使う。

## 役割・なぜ必要か
- Express は「ミドルウェアでリクエストを処理する箱」だけを提供する最小フレームワーク。**永続化は守備範囲外**。
- だから「Rails ならモデルがある」感覚で来ると、`User.find` 相当が**最初から存在しない**ことに戸惑う。まず ORM/ドライバを選ぶのが出発点。
- 選んだ後は **接続管理（コネクションプール）・クエリ・トランザクション・N+1 対策**を自分で組む。これらを薄いリポジトリ層に隔離するのが定番。

---

## まず選ぶ（内蔵なし前提）
| 用途 | 代表的な選択肢 | 特徴 |
|------|----------------|------|
| SQL（型安全・人気） | **Prisma** | スキーマファイル＋自動生成で型安全。学習が速い |
| SQL（成熟・自由度） | Sequelize / TypeORM | 歴史が長い。生 SQL も書きやすい |
| MongoDB | **Mongoose** | スキーマ／バリデーション付きの定番 ODM |
| 生 SQL を直接 | `pg` / `mysql2` ＋ クエリビルダ（Kysely 等） | ORM を挟まず薄く扱う |

## Prisma の基本（SQL 系・型安全）
```prisma
// prisma/schema.prisma
model User {
  id    Int    @id @default(autoincrement())
  email String @unique
  posts Post[]
}
model Post {
  id       Int    @id @default(autoincrement())
  title    String
  author   User   @relation(fields: [authorId], references: [id])
  authorId Int
}
```
```js
// db.js: PrismaClient はアプリで1個だけ作って使い回す（プール内蔵）
const { PrismaClient } = require('@prisma/client')
const prisma = new PrismaClient()
module.exports = prisma
```
```js
const prisma = require('./db')
const user = await prisma.user.findUnique({ where: { id } }) // await 必須
```

## 接続・コネクションプール
```js
// NG: リクエストごとに new すると接続が枯渇する
app.get('/x', async (req, res) => {
  const client = new PrismaClient() // ← 毎回作るな
})

// OK: 単一インスタンスを共有（Prisma/Mongoose とも接続プールを内蔵）
const prisma = require('./db') // 起動時に1個
```
- DB への TCP 接続は有限。**プール上限 × インスタンス数** が DB の最大接続を超えると「too many connections」。
- 起動時に接続、終了時に切断する（グレースフルシャットダウン）。
```js
process.on('SIGTERM', async () => {
  await prisma.$disconnect()
  process.exit(0)
})
```

## リポジトリ層に分離する（定番）
```js
// repositories/userRepository.js : DB アクセスをここに閉じ込める
module.exports = {
  findById: (id) => prisma.user.findUnique({ where: { id } }),
  create: (data) => prisma.user.create({ data }),
  list: ({ skip = 0, take = 20 }) => prisma.user.findMany({ skip, take }),
}
```
- ハンドラ（ルート）から ORM を直接叩かず、**リポジトリ経由**にする。
- 利点: テストでモックしやすい・ORM を差し替えやすい・SQL が一箇所に集まる（[project_structure.md](./project_structure.md)）。

## N+1 問題とは
一覧で関連を1件ずつ引き、SQL が `1 + N` 回走る性能劣化。
```js
// NG: ループ内で都度クエリ → N+1
for (const post of posts) {
  post.author = await prisma.user.findUnique({ where: { id: post.authorId } })
}

// OK: include で関連をまとめて取得（JOIN/IN にまとまる）
const posts = await prisma.post.findMany({ include: { author: true } })

// Mongoose なら populate
const posts = await Post.find().populate('author')
```
**対策の選択肢**:
- Prisma **`include`**（関連を丸ごと）vs **`select`**（必要フィールドだけ・軽量）。ネストも可 `include: { author: { include: { profile: true } } }`。
- 件数だけなら `_count`（`include: { _count: { select: { comments: true } } }`）。
- 生SQLなら `WHERE id IN (...)` でまとめて1クエリ。Mongooseは `populate`（N+1を1クエリ化）、読み取りは `.lean()` で軽いプレーンオブジェクト。

## トランザクション
```js
// どれか失敗で全部ロールバック
await prisma.$transaction(async (tx) => {
  await tx.order.create({ data: order })
  await tx.stock.update({ where: { id }, data: { count: { decrement: 1 } } })
})
```

## ハマりどころ / アンチパターン
- **内蔵 ORM があると思い込む**：まず選定が必要。標準は無い。
- **コネクション管理（プール枯渇）**：リクエストごとに `new` する／切断しないと枯渇。**単一インスタンス共有**が基本。
- **N+1**：ループ内クエリ → `include`/`populate`/`in` でまとめる。
- **await 漏れ**：`prisma.user.findMany()` を `await` せず使うと Promise が返り、データが空/型崩れ（[async_patterns.md](./async_patterns.md)）。
- **マイグレーション忘れ**：スキーマ変更したら `prisma migrate` 等を必ず流す。
- **生 SQL の文字列連結**：SQL インジェクション。必ずパラメータ化／ORM のクエリを使う。
- **接続情報のハードコード**：`DATABASE_URL` は環境変数で（[config_env.md](./config_env.md) 想定）。

## ORM 特有の現象と対策（N+1以外）
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| **コネクションプール枯渇** | `PrismaClient` をリクエスト毎/モジュール乱立で複数生成すると接続が枯渇。特に**サーバレス**は関数インスタンス毎に接続を張り上限に達する | **シングルトンを1個**だけ生成して共有。サーバレスは `connection_limit` を調整、Prisma Accelerate / Data Proxy で接続をプール |
| ページング無しの `findMany` | 全件取得で重い・メモリ増 | `take`/`skip`（オフセット）か **cursor ページング**（`cursor`/`take`）。一覧は必ず上限 |
| `$transaction` の使い分け | 配列バッチ（独立操作の原子化）とインタラクティブ（コールバックでロジック込み）は用途が違う | 独立操作は配列 `$transaction([...])`、条件分岐を挟むなら `$transaction(async (tx) => {...})` |
| インデックス欠如 | 検索/結合カラムにindexが無いと全表スキャン | schema に `@@index([...])` / `@unique`。マイグレーションを流す |
| 複雑クエリがORMで書けない | 集計・ウィンドウ関数など | `$queryRaw`（パラメータ化必須・型は手当て） |
| Mongoose の罠 | `populate` のN+1、スキーマ外フィールドの混入、ドキュメントが重い | `populate` でまとめ、読み取りは `.lean()`、`strict` でスキーマ外を弾く |
| 浮いたPromise/await漏れ | `await` 忘れで未解決Promiseを使う | DB呼び出しは必ず `await`（土台の罠 → [../../../../フロントエンド/javascript/落とし穴.md](../../../../フロントエンド/javascript/落とし穴.md)） |

## 関連
[project_structure.md](./project_structure.md)
