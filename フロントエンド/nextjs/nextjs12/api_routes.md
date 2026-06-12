# API ルート（pages/api）（Next.js 12・Pages Router）

## ひとことで言うと
`pages/api/` 配下のファイルが **HTTPエンドポイント**になり、`export default function handler(req, res)` の `req.method` で分岐して `res` で応答を返す——Next.js に内蔵されたサーバ関数置き場。

## 役割・なぜ必要か
- フロントと同じプロジェクトに **小さなバックエンド**を同居できる（別サーバを立てずに、フォーム送信先・Webhook受け・DB読み書き・外部API中継を置ける）。
- これらはサーバでだけ動くので、**秘密鍵やDB接続をクライアントに晒さず**呼べる。`getServerSideProps` から内部fetchする先や、クライアントの SWR/fetch の取得先になる。
- ファイル＝URLの規則はページと同じ。`pages/api/users/[id].ts` が `/api/users/123` になる。

## 基本の書き方（コード）

```ts
// pages/api/hello.ts → "/api/hello"
import type { NextApiRequest, NextApiResponse } from "next";

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  res.status(200).json({ message: "hello" });
}
```

```ts
// pages/api/users/index.ts → "/api/users"  メソッドで分岐
import type { NextApiRequest, NextApiResponse } from "next";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === "GET") {
    const users = await db.users.findMany();
    return res.status(200).json(users);
  }
  if (req.method === "POST") {
    const { name } = req.body; // JSONは自動でパースされ req.body に入る
    if (!name) return res.status(400).json({ error: "name は必須" });
    const created = await db.users.create({ data: { name } });
    return res.status(201).json(created);
  }
  res.setHeader("Allow", ["GET", "POST"]);
  return res.status(405).json({ error: "Method Not Allowed" });
}
```

```ts
// pages/api/users/[id].ts → "/api/users/123"  動的API
import type { NextApiRequest, NextApiResponse } from "next";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { id } = req.query; // "123"（string | string[]）
  const user = await db.users.findUnique({ where: { id: String(id) } });
  if (!user) return res.status(404).json({ error: "Not Found" });
  return res.status(200).json(user);
}
```

```ts
// 設定例：bodyParser の上限変更や無効化（Webhook の生body検証など）
export const config = {
  api: {
    bodyParser: { sizeLimit: "1mb" },
    // bodyParser: false, // ← 生body を自分で読みたいとき（Stripe等の署名検証）
  },
};
```

呼び出し側（クライアント）。
```ts
const res = await fetch("/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "太郎" }),
});
const user = await res.json();
```

## 実務での使い方・定番パターン
- **メソッド分岐**：1ファイルで `GET`/`POST`/`PUT`/`DELETE` を `req.method` で振り分け、未対応は `405` + `Allow` ヘッダ。
- **入力検証**：`req.body`/`req.query` を信用せず、zod 等で検証してから処理（→ 不正値で落とさない）。
- **認証**：cookie/JWT を `req.cookies`/`req.headers.authorization` から取り、共通の `withAuth` ラッパーで保護。
- **外部API中継（BFF）**：秘密鍵が要る外部APIをここで呼び、クライアントには鍵を見せない。
- **Webhook受け**：署名検証が要るものは `bodyParser: false` にして生bodyを読む。

## ハマりどころ / アンチパターン
- **App Router の `route.ts` と混同**：App Router は `app/api/x/route.ts` で `Request`/`Response`（Web標準）と `export async function GET()`。Pages Router は `(req, res)` の Node 形式（[../nextjs15/](../nextjs15/)）。別物。
- **`res` の二重送信**：`res.json()` の後にさらに `res.send()` 等を呼ぶと `Cannot set headers after they are sent`。各分岐は `return res...` で抜ける。
- **`req.body` が空/文字列**：`Content-Type: application/json` を付けないとパースされない。送信側のヘッダを確認。
- **メソッド未指定で全部処理**：`req.method` を見ずに全リクエストを同じ扱いにすると、想定外メソッドで誤動作。`405` を返す。
- **重い同期処理/長時間処理**：サーバレス前提だとタイムアウトする。重い処理はキュー/バックグラウンドへ。
- **秘密をクライアントに返す**：DBの生レコードをそのまま `json()` で返してハッシュやトークンを漏らさない。返す前に整形。

## 関連
[data_fetching.md](./data_fetching.md) / [middleware.md](./middleware.md) / [routing.md](./routing.md) / [pitfalls.md](./pitfalls.md)
