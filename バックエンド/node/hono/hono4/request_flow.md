# リクエストの流れ・各層は何を返すか（Hono v4）

## ひとことで言うと
1リクエストが **Middleware → Handler → Service** と降り、**Service がデータを上げてきて、handler が `c.json()` で返す**。全層が Context `c` を共有し、入出力は `c`（`c.req` 入力 / `c.json()` 出力）で行う。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（POSTなら request body = JSON）
   ▼
[Middleware]  onion: 前処理 → await next() → 後処理（cors/jwt/logger等）
   │
   ▼
[Handler(c)]  c.req.json() で入力取得／Service を呼ぶ（決まった構造はない、自前で作る）
   │
   ▼
[Service]     （自前）業務ロジック／DB を呼ぶ
   │
   ▼
[DB(Drizzle/Prisma/D1)]  DBアクセス（内蔵なし）
   │
   ▼
  DB ──→ レコード(オブジェクト) を返す ─┐
   ▲                                    │
[Service] が オブジェクト/DTO を handler に返す
   ▲
[Handler] が return c.json(...) でレスポンスを返す
   │ レスポンス（response body = JSON）
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **Middleware** | `Context c`, `next` | `await next()`で続行 / `c`で返す（拒否） |
| **Handler** | `Context c`（`c.req`で入力） | **`c.json()` / `c.text()`**（Responseを返す） |
| **Service**（自前） | DTO / 値 | **オブジェクト / DTO** |
| **DB層** | id / 条件 / 保存データ | **レコードオブジェクト**（DBから） |

- Honoは Web標準の `Request`/`Response` ベース。handlerは `c.json()` 等が返す `Response` を `return` する（Expressの `res.json` と違い戻り値）。
- Service/DBの層は自前で切る。型安全RPCを使うとhandlerの戻り型がクライアントへ伝播する。

## コードで通して見る
```ts
// 1) Handler：c.req.json()で入力 → Serviceを呼ぶ → c.json()を return
app.post("/posts", async (c) => {
  const body = await c.req.json();              // 入力取得
  const post = await postService.create(body);  // Serviceがオブジェクトを返す
  return c.json({ id: post.id, title: post.title }); // Responseを返す
});

// zod-validatorで検証する形（境界を固める）
app.post("/posts", zValidator("json", PostSchema), async (c) => {
  const data = c.req.valid("json");             // 検証済みデータ
  const post = await postService.create(data);
  return c.json(post);
});

// 2) Service（自前）：業務処理 → DB呼び出し → オブジェクトを返す
export const postService = {
  async create(data: PostInput) {
    return db.insert(posts).values(data).returning(); // DBがレコードを返す
  },
};
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：DB→Service はレコードオブジェクト、Service→handler はオブジェクト/DTO、handler→クライアントは `c.json()`。
- **入力検証は zod-validator**：`c.req.valid()` で型付きの検証済みデータを取る。→ [validation.md](./validation.md)
- **横断処理は組込ミドルウェア**：cors / jwt / logger をルートに付与。→ [middleware.md](./middleware.md)
- **型安全RPCで戻り型を伝播**：ルートをチェーン定義し `hc<AppType>()` でクライアント型付け。→ [rpc.md](./rpc.md)

## ハマりどころ / アンチパターン
- **handlerで `c.json()` を `return` し忘れる**：レスポンスが返らない。必ず `return`。→ [context.md](./context.md)
- **handlerに業務とDBを全部書く**：肥大化。Serviceに切り出す。
- **`c.req.json()` を二重に読む**：ボディは一度しか読めない。変数に保持する。→ [pitfalls.md](./pitfalls.md)
- **ランタイム固有APIに依存**：`c.env` のバインディングはWorkers前提。移植時に注意。→ [runtimes.md](./runtimes.md)

## 関連
[routing.md](./routing.md) / [context.md](./context.md) / [middleware.md](./middleware.md) / [validation.md](./validation.md) / [rpc.md](./rpc.md) / [database.md](./database.md)
