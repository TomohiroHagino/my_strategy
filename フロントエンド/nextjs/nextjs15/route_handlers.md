# Route Handlers（API：route.ts）（Next.js 15）

## ひとことで言うと
`app/` 配下に **`route.ts`** という名前で置くと、その URL が **API エンドポイント**になる仕組み。`GET` / `POST` などの **HTTP メソッド名で関数を export** し、引数・戻り値は **Web 標準の `Request` / `Response`** を使う。Pages Router の `pages/api/*` に相当する、App Router 版の API。

## 役割・なぜ必要か
- 画面（`page.tsx`）を返すのではなく、**JSON や任意のレスポンスを返す**ためのファイル。外部サービスからの Webhook 受信、サードパーティ API のプロキシ、ファイル配信、認証コールバックなどに使う。
- 一方、フォーム送信や自アプリ内の DB 更新は **Server Actions** のほうが簡潔なことが多い。→ [server_actions.md](./server_actions.md)。Route Handlers は「**外部に公開する HTTP インターフェース**が欲しい時」に選ぶ。
- Web 標準（`Request`/`Response`/`Headers`）に乗っているので、Edge ランタイムや他環境へも移植しやすい。

## 基本の書き方（コード）
```ts
// app/api/todos/route.ts  ← ファイル名は必ず route.ts
import { NextResponse } from "next/server";
import { z } from "zod";
import { db } from "@/lib/db";

// GET /api/todos
export async function GET(req: Request) {
  // クエリは Web 標準の URL から取り出す
  const { searchParams } = new URL(req.url);
  const limit = Number(searchParams.get("limit") ?? 20);

  const todos = await db.todo.findMany({ take: Math.min(limit, 100) });
  // NextResponse.json で JSON レスポンスを返す（ヘッダ／ステータス指定も可）
  return NextResponse.json({ data: todos });
}

const CreateTodo = z.object({ title: z.string().min(1).max(100) });

// POST /api/todos
export async function POST(req: Request) {
  // ボディは Web 標準の req.json() で読む
  const body = await req.json();
  const parsed = CreateTodo.safeParse(body); // 入力検証は必須
  if (!parsed.success) {
    return NextResponse.json({ error: "invalid" }, { status: 400 });
  }

  const todo = await db.todo.create({ data: parsed.data });
  return NextResponse.json({ data: todo }, { status: 201 });
}
```

```ts
// app/api/todos/[id]/route.ts（動的セグメント）
// Next 15 では params が Promise なので await する
export async function DELETE(
  req: Request,
  { params }: { params: Promise<{ id: string }> },
) {
  const { id } = await params;
  await db.todo.delete({ where: { id } });
  return new Response(null, { status: 204 }); // 標準 Response も使える
}
```

## 実務での使い方・定番パターン
- **メソッドごとに関数を export**：`GET` `POST` `PUT` `PATCH` `DELETE` `HEAD` `OPTIONS`。export していないメソッドは Next が自動で 405 を返す。
- **`NextResponse.json(data, { status })`**：JSON を返す定番。ステータスやヘッダもここで設定する。リダイレクトは `NextResponse.redirect(url)`、Cookie は `NextResponse` の `cookies` API。
- **動的ルート**：`app/api/items/[id]/route.ts` の `params` から `id` を受け取る（**Next 15 では `params` が `Promise`** なので `await`）。
- **キャッシュ制御**：`GET` は条件次第で静的化される。常に最新にしたいなら `export const dynamic = "force-dynamic"`、Edge で動かすなら `export const runtime = "edge"`。
- **Webhook 受信**：外部署名の検証には**生のボディ**が要るので `await req.text()` で取り、検証後に `JSON.parse` する。
- **レスポンス形式を統一**：`{ data, error }` のような共通エンベロープにすると、クライアント側の扱いが楽。

## ハマりどころ / アンチパターン
- **ファイル名が `route.ts` でない**：`api.ts` や `index.ts` では認識されない。**必ず `route.ts`（または `route.js`）**。同じフォルダに `page.tsx` と `route.ts` を同居させるのも不可。
- **メソッド名以外で export**：`export async function handler()` のような独自名は無効。**HTTP メソッド名そのもの**で export する。
- **`req`/`res` を Express 流で書く**：Next の Route Handler は **Web 標準の `Request`/`Response`**。`res.json(...)` ではなく `return Response`/`NextResponse` を返す。`res` 引数は存在しない。
- **入力検証・認可の欠落**：公開エンドポイントなので、ボディ／クエリは zod 等で検証し、保護したい操作はトークンやセッションを確認する。Server Action 同様、誰でも叩ける前提で書く。
- **Next 15 で `params` を同期アクセス**：`params.id` を直接読むと警告／エラー。`await params` してから使う。
- **重い処理を `GET` に詰める**：副作用（DB 更新）は `GET` ではなく `POST`/`PUT`/`DELETE` に。`GET` はキャッシュされうるので変更処理に向かない。

## 関連
[server_actions.md](./server_actions.md) / [middleware.md](./middleware.md)
