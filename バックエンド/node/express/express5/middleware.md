# ミドルウェア（Middleware）（Express 5）

## ひとことで言うと
**`(req, res, next) => {}` という形の関数**で、リクエストとレスポンスの「間」に挟まって処理を行う部品。`app.use()` で登録すると、リクエストが上から順にこの関数の列を通っていく。Express の処理はほぼ「ミドルウェアの連鎖」で出来ている＝**Expressの核**。

## 役割・なぜ必要か
- ログ出力・認証・ボディ解析・CORS・エラー整形など、**全ルート（または一部）に共通する前処理／後処理**を1か所にまとめられる。
- 各ミドルウェアは「自分の仕事をして、`next()` を呼んで次へ渡す」だけ。小さな関数を**順番に並べて積み上げる**ことで、複雑な処理を分割できる。
- ルートハンドラ（`app.get(...)` の最後の関数）も、実態は「最後のミドルウェア」。つまり全部が同じ仕組みで動く。

## 基本の書き方（コード）
```js
const express = require("express");
const app = express();

// 1) グローバル：全リクエストが通る（登録順＝実行順）
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();                       // ← これを呼ばないと先に進まない
});

// 2) 組込ミドルウェア：JSON ボディを req.body に展開
app.use(express.json());                       // Content-Type: application/json
app.use(express.urlencoded({ extended: true })); // フォーム送信(x-www-form-urlencoded)
app.use(express.static("public"));             // public/ を静的配信

// 3) ルート単位：このルートだけに挟む認証
const requireAuth = (req, res, next) => {
  if (!req.headers.authorization) {
    return res.status(401).json({ error: "認証が必要です" }); // ここで終了（next 呼ばない）
  }
  next();                       // OK なら次へ
};

app.get("/me", requireAuth, (req, res) => {
  res.json({ user: "alice" });  // 最後のミドルウェア＝ハンドラ
});

// 4) エラー用ミドルウェア：引数が「4個」だと Express がエラー用と認識する
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: "Internal Server Error" });
});

app.listen(3000);
```

## 実務での使い方・定番パターン
- **3つの登録単位を使い分ける**
  - グローバル：`app.use(fn)` … 全リクエスト共通（ログ・ボディ解析・CORS）。
  - ルート単位：`app.get("/x", mw, handler)` … 引数に複数並べると左から実行。認証・権限チェックに多用。
  - Router 単位：`router.use(mw)` … 特定パス配下だけに適用（`/admin` 以下は全て認証、など）。
- **組込ミドルウェア**（Express 同梱）
  - `express.json()` … JSON ボディを解析。**これが無いと `req.body` は `undefined`**。
  - `express.urlencoded({ extended: true })` … HTMLフォーム送信を解析。
  - `express.static("public")` … 画像・CSS など静的ファイルを配信。
- **サードパーティ**（`npm i` して `app.use`）
  - `cors` … CORS ヘッダ付与（別オリジンからのAPI呼び出し許可）。
  - `helmet` … セキュリティ用HTTPヘッダを自動付与。
  - `morgan` … アクセスログ整形（`app.use(morgan("dev"))`）。
- **順序が全て**：`express.json()` はルート定義より「前」、エラーハンドラは「一番最後」。`helmet`/`cors` は早い段階で。

```js
const cors = require("cors");
const helmet = require("helmet");
const morgan = require("morgan");

app.use(helmet());            // セキュリティヘッダ（早めに）
app.use(cors());              // CORS（ルートより前）
app.use(morgan("dev"));       // アクセスログ
app.use(express.json());      // ボディ解析（ルートより前）
// …この後にルート定義…
// …一番最後にエラーハンドラ…
```

## ハマりどころ / アンチパターン
- **`next()` 呼び忘れでリクエストがハング**：最頻出。`res.json()` などで応答も返さず `next()` も呼ばないと、クライアントは永久に待つ（タイムアウト）。「応答を返す」か「`next()` を呼ぶ」か、**必ずどちらか**を実行する。
- **`next()` と応答の二重実行**：`res.json()` の後に `next()` を呼ぶと、後続が再び `res` を触り `Cannot set headers after they are sent` エラー。応答したら `return` で抜ける。
- **登録順ミス**：`express.json()` をルートより後に書くと、そのルートでは `req.body` が `undefined`。ミドルウェアは「上から順」。
- **エラー用なのに引数3個**：`(err, req, res, next)` の**4個でないと**エラー用と認識されず、ただの普通のミドルウェア扱いになる。→ [error_handling.md](./error_handling.md)
- **重い同期処理をミドルウェアに直書き**：イベントループを止めて全リクエストが詰まる。非同期化／別処理へ。→ [async_patterns.md](./async_patterns.md)
- **`app.use("/path", mw)` のパスマッチ誤解**：`app.use` のパスは「前方一致」。`/users` 指定で `/users/1` も通る。

## 関連
[request_response.md](./request_response.md) / [error_handling.md](./error_handling.md) / [routing.md](./routing.md)
