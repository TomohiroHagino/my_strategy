# リクエストの流れ・各層は何を返すか（Express 5）

## ひとことで言うと
1リクエストが **Middleware(app.use) → Route handler((req,res)) → (Controller/Service/Model)** と降り、**Model がデータを上げてきて、handler が `res.json`/`res.render` で返す**。Expressは層を自前で作るのが前提。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（POSTなら request body = JSON）
   ▼
[Middleware(app.use)]  express.json()/認証/ログなど横断処理（next()で次へ）
   │
   ▼
[Route handler((req,res))]  req.body で入力取得／Controller等を呼ぶ
   │
   ▼
[Controller]  （自前で作る層）入力検証・Service呼び出し
   │
   ▼
[Service]     （自前）業務ロジック／Model を呼ぶ
   │
   ▼
[Model(ORM)]  （Prisma/Sequelize等）DBアクセス
   │
   ▼
  DB ──→ レコード(オブジェクト) を返す ─┐
   ▲                                    │
[Model] が オブジェクト を Service に返す
   ▲
[Service]   が オブジェクト/DTO を handler に返す
   ▲
[handler]   が res.json(JSON) or res.render(HTML) で返す
   │ レスポンス（response body = JSON or HTML）
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **Middleware** | `(req, res, next)` | `next()`で続行 / `res`で返す（拒否） |
| **Route handler** | `(req, res)`（`req.body`で入力） | `res.json` / `res.render`（戻り値なし） |
| **Controller**（自前） | req/値 | Serviceを呼びhandlerへ結果 |
| **Service**（自前） | DTO / 値 | **オブジェクト / DTO** |
| **Model(ORM)** | id / 条件 / 保存データ | **レコードオブジェクト**（DBから） |

- Expressは無規約。Controller/Service/Modelの層は自分で切る（規約はフレームワークが与えない）。
- handlerは値を返さず、`res.json`/`res.render` でレスポンスへ書き込む。Express 5は async関数のrejectを自動でエラーミドルウェアへ転送する。

## コードで通して見る
```js
// 1) Route handler：req.bodyで入力 → Serviceを呼ぶ → res.jsonで返す
app.post("/posts", async (req, res) => {
  const post = await postService.create(req.body); // Serviceがオブジェクトを返す
  res.json({ id: post.id, title: post.title });     // JSONで返す
});
// ↑ Express 5: async内でthrowすればエラーミドルウェアへ自動転送（4はnext(err)が必要だった）

// 2) Service（自前）：業務処理 → Model呼び出し → オブジェクトを返す
export const postService = {
  async create(data) {
    return prisma.post.create({ data });            // Modelがレコードを返す
  },
};

// 3) エラーミドルウェアは最後に登録（4引数で判別）
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message });
});
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：Model→Service はレコードオブジェクト、Service→handler はオブジェクト/DTO、handler→クライアントは `res.json`/`res.render`。
- **層は自分で切る**：routes / controllers / services / models のディレクトリを作る。→ [project_structure.md](./project_structure.md)
- **入力検証はミドルウェアで**：zod/express-validatorで境界を固める。→ [validation.md](./validation.md)
- **エラーは集約**：handlerでthrow→末尾のエラーミドルウェアで一括処理。→ [error_handling.md](./error_handling.md)

## ハマりどころ / アンチパターン
- **handlerに業務とDBを全部書く**：肥大化。Service/Modelに切り出す。
- **`res` を二重送信**：`res.json` の後に処理を続けると `Cannot set headers after sent`。返したら `return`。→ [pitfalls.md](./pitfalls.md)
- **エラーミドルウェアを途中に置く**：4引数のエラーハンドラは最後に登録する。→ [error_handling.md](./error_handling.md)
- **Express 5のルート記法変更で躓く**：`*`→`/*splat`、`:id?`→`{/:id}`。4からの移行で要注意。→ [routing.md](./routing.md)

## 関連
[middleware.md](./middleware.md) / [routing.md](./routing.md) / [request_response.md](./request_response.md) / [project_structure.md](./project_structure.md) / [error_handling.md](./error_handling.md) / [validation.md](./validation.md)
