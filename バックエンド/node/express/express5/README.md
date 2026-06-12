# Express 5 実務リファレンス（索引）

> **この版 = Express 5（Node.js 18+）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。

## この版のポイント（Express 5 で変わったこと）
- **async ハンドラの例外を自動でエラーハンドラへ転送**（4では `next(err)` を自分で呼ぶ/ラップが必要だった。5は `async` 関数が reject すると自動で error middleware へ）。
- **ルート記法の刷新（path-to-regexp v8）**：`*` は名前付き `/*splat`、オプションは `/:id?` → `{/:id}`、ルート内の生正規表現は制限。**4からの移行で一番引っかかる所**。
- **Node 18+ 必須**。古いNodeサポート終了。
- **非推奨APIの削除**：`app.del()`→`app.delete()`、`res.json(obj, status)`→`res.status(x).json(obj)`、`res.sendfile`→`res.sendFile`、`req.param(name)` 廃止。
- ボディ解析は内蔵（`express.json()` / `express.urlencoded()`）。

## リクエストの流れ（全体像）
```
リクエスト → ミドルウェア① → ミドルウェア② → … → ルートハンドラ → レスポンス
            (app.use / 各 next() で次へ)        途中で res を返すか next(err) でエラー段へ
```
> Expressは「**ミドルウェアの列を順に通る**」だけ。これが全ての土台。

## 項目（各ファイルへ）

### はじめに / コア
- [getting_started.md](./getting_started.md) … 始め方（npm / app.listen / nodemon）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [project_structure.md](./project_structure.md) … プロジェクト構成（無規約をどう整えるか）
- [routing.md](./routing.md) … ルーティング（app.get / Router / パラメータ）とは
- [middleware.md](./middleware.md) … ミドルウェアとは（Expressの核）
- [request_response.md](./request_response.md) … req / res とは

### 処理・データ
- [error_handling.md](./error_handling.md) … エラーハンドリング（error middleware）とは
- [async_patterns.md](./async_patterns.md) … async/await とイベントループとは
- [database.md](./database.md) … DB（Prisma / Sequelize / Mongoose、内蔵なし）とは
- [validation.md](./validation.md) … バリデーション（zod / express-validator）とは

### 認証・安全・設定・運用
- [auth.md](./auth.md) … 認証（JWT / セッション / Passport）とは
- [security.md](./security.md) … セキュリティ（helmet / CORS / レート制限）とは
- [config_env.md](./config_env.md) … 設定（dotenv / process.env）とは
- [templating.md](./templating.md) … テンプレート（res.render / EJS）※API主体なら任意
- [testing.md](./testing.md) … テスト（Jest / Vitest ＋ supertest）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Express 5）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
