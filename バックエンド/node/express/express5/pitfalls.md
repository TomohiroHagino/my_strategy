# 実務でハマる罠まとめ（Pitfalls）（Express 5）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Express は「ミドルウェアの列を通すだけ」のミニマルなフレームワーク。自由度が高い分、規約が無く事故りやすい。症状から該当箇所へ素早く飛ぶための索引。

## ミドルウェア / 制御フロー
- **`next()` 呼び忘れでハング**：ミドルウェアで `res` も返さず `next()` も呼ばないと、その先へ進めずリクエストが応答せず固まる。必ず「応答する」か「`next()` する」。→ [middleware.md](./middleware.md)
- **ミドルウェアの順序ミス**：`app.use` は登録順に実行。`express.json()` や認証を該当ルートより**後**に置くと効かない。順序が全て。→ [middleware.md](./middleware.md)
- **二重応答**：分岐後の `return` 忘れで `res.json` を2回呼び `Cannot set headers after they are sent`。応答後は必ず `return`。→ [request_response.md](./request_response.md)

## 非同期 / パフォーマンス
- **イベントループを塞ぐと全体停止**：同期で重い処理（巨大ループ・同期I/O・`JSON.parse` 巨大データ）を回すと、シングルスレッドゆえ全リクエストが止まる。重い処理は非同期化／別プロセス／ワーカーへ。→ [async_patterns.md](./async_patterns.md)
- **未処理の Promise reject**：await し忘れ・catch 漏れで `unhandledRejection`。Express 5 は async ハンドラの reject を自動でエラー段へ送るが、ハンドラ外の浮いた Promise は拾われない。→ [async_patterns.md](./async_patterns.md) / [error_handling.md](./error_handling.md)

## リクエスト / レスポンス
- **`express.json()` 無しで `req.body` が undefined**：ボディ解析ミドルウェアを入れないと JSON ボディが読めない。`app.use(express.json())` を忘れない（フォームは `express.urlencoded()`）。→ [request_response.md](./request_response.md)

## Express 5 の記法変更（4からの移行で要注意）
- **ルート記法の刷新（path-to-regexp v8）**：`*` は `/*splat`、オプションは `/:id?` → `{/:id}`、ルート内の生正規表現は制限。4の書き方のままだと起動時にルートが壊れる。→ [routing.md](./routing.md)
- **削除された API**：`app.del()`→`app.delete()`、`res.json(obj, status)`→`res.status(x).json(obj)`、`res.sendfile`→`res.sendFile`、`req.param()` 廃止。→ [routing.md](./routing.md) / [error_handling.md](./error_handling.md)

## 設計 / 構成
- **無規約で構造が肥大**：Express は構成を強制しない。1ファイルに全部書くと巨大化。routes / controllers / services / middlewares に層分けする。→ [project_structure.md](./project_structure.md)

## セキュリティ / 設定
- **helmet / CORS 未設定**：素の Express はセキュリティヘッダも CORS 制御も無い。`helmet()` でヘッダ付与、`cors()` でオリジン制御、レート制限も入れる。→ [security.md](./security.md)
- **`.env` をコミット**：DB パスワードや API キー入りの `.env` を Git に上げる事故。`.gitignore` 必須、漏れたら即ローテート。→ [config_env.md](./config_env.md)
- **DB 内蔵なし＝自分で選ぶ**：Express に ORM/DB は無い。Prisma / Sequelize / Mongoose 等を自分で選定・接続管理する（接続プール・切断処理も自前）。→ [database.md](./database.md)

## 関連
[middleware.md](./middleware.md) / [routing.md](./routing.md) / [async_patterns.md](./async_patterns.md) / [request_response.md](./request_response.md) / [error_handling.md](./error_handling.md) / [project_structure.md](./project_structure.md) / [security.md](./security.md) / [config_env.md](./config_env.md) / [database.md](./database.md)
