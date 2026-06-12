# リクエストの流れ・各層は何を返すか（Next.js 15・App Router）

## ひとことで言うと
HTTPリクエストが **middleware → ルート（`page.tsx` or `route.ts`）→ Server Component の async fetch/DB** と降り、**HTML（RSCストリーム）が返る**。「どの部分が何を受け取り、何を返すか」を1枚で俯瞰する。フルスタック（サーバ実行あり）なので「層」がある。

## 全体の流れ（図）
```
ブラウザ
   │ HTTP リクエスト（GET /users/123 など）
   ▼
[middleware.ts]   Edge で先に走る（認証・リダイレクト・ヘッダ）  受:NextRequest → 返:NextResponse
   │ （pass）
   ▼
[ルーティング]    URL → app/ のファイルを解決（page.tsx か route.ts）
   │
   ├─ 画面の場合 ─────────────────────────────────┐
   │   ▼                                          │
   │ [Server Component (page.tsx)]  async で fetch/DB を直接叩く
   │   │ 受:params/searchParams → 返:JSX
   │   ▼
   │ [RSC → HTML 直列化]   サーバでHTMLにしてストリーム送信
   │   │
   │   ▼
   │  HTML が返る（"use client" の所だけブラウザで hydrate）
   │
   └─ API の場合 ─────────────────────────────────┐
       ▼                                          │
     [Route Handler (route.ts)]  GET/POST 関数      受:Request → 返:Response(JSON)
   ▼
ブラウザ（HTML描画 or JSON受信）

（データ変更）フォーム submit → [Server Action "use server"] サーバ関数実行 → 返:結果（再検証）
```

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 実行場所 |
|---|---|---|---|
| **middleware.ts** | `NextRequest`（URL/cookie/header） | `NextResponse`（通過 / redirect / rewrite） | Edge |
| **Server Component（page.tsx）** | `params` / `searchParams` | **JSX**（→サーバでHTML化） | サーバ |
| **RSC → HTML 直列化** | コンポーネントツリー | **HTMLストリーム（RSCペイロード）** | サーバ |
| **Route Handler（route.ts）** | `Request`（API呼び出し） | **`Response`（JSON等）** | サーバ |
| **Server Action（"use server"）** | フォームデータ / 引数 | **処理結果**（＋キャッシュ再検証） | サーバ |
| **Client Component（"use client"）** | props | DOM（ブラウザで hydrate 後に対話） | ブラウザ |

- **Server Component が既定**。データ取得（fetch/DB）をコンポーネント内で `async` のまま書ける。
- **`"use client"` を付けた部分だけ**ブラウザに送られ、hydrate されて対話可能になる。

## コードで通して見る
```ts
// 1) middleware.ts：Edge で全リクエストの前段に走る
import { NextResponse, type NextRequest } from "next/server";
export function middleware(req: NextRequest) {
  if (!req.cookies.get("session")) {
    return NextResponse.redirect(new URL("/login", req.url)); // 返り＝NextResponse
  }
  return NextResponse.next();                                  // 通過
}
export const config = { matcher: ["/users/:path*"] };

// 2) Server Component（app/users/[id]/page.tsx）：async で直接 fetch → JSX を返す
export default async function UserPage({ params }: { params: { id: string } }) {
  const res = await fetch(`https://api.example.com/users/${params.id}`); // サーバで取得
  const user = await res.json();
  return <h1>{user.name}</h1>;            // 返り＝JSX（サーバでHTML化されてストリーム）
}

// 3) Route Handler（app/api/users/route.ts）：Request → Response(JSON)
import { NextResponse } from "next/server";
export async function POST(req: Request) {
  const body = await req.json();          // 受け＝Request
  const created = await db.user.create({ data: body });
  return NextResponse.json(created);      // 返り＝Response(JSON)
}

// 4) Server Action（変更処理）："use server" でサーバ実行
export async function createUser(formData: FormData) {
  "use server";
  await db.user.create({ data: { name: String(formData.get("name")) } });
  revalidatePath("/users");               // 返り＝結果（＋再検証）
}
```

## 実務での使い方・定番パターン
- **画面はまず Server Component**：fetch/DBをコンポーネント内で完結させ、HTMLでクライアントに返す。対話が要る所だけ `"use client"` の子に切る。
- **API（外部やクライアント取得用）は Route Handler**：`route.ts` の `GET`/`POST` で `Response` を返す。
- **変更（作成・更新・削除）は Server Action**：フォーム送信からサーバ関数を直接呼び、`revalidatePath`/`revalidateTag` でキャッシュを更新。→ [server_actions.md](./server_actions.md)
- **境界を意識**：Server↔Client の境界で渡せるのはシリアライズ可能な props のみ。

## ハマりどころ / アンチパターン
- **Server Component で `useState`/`useEffect` を使う**：それらは Client 専用。`"use client"` が要る。→ [server_client_components.md](./server_client_components.md)
- **middleware に重い処理を書く**：Edge は制約あり（Node API 不可・短時間）。認証チェック等に留める。→ [middleware.md](./middleware.md)
- **fetch のキャッシュ挙動を誤解**：Next 15 で既定が見直された。`cache`/`revalidate` を明示。→ [caching.md](./caching.md)
- **Client へ大きなオブジェクトを props で渡す**：直列化コスト・バンドル肥大。サーバで絞ってから渡す。

## 関連
[middleware.md](./middleware.md) / [server_client_components.md](./server_client_components.md) / [data_fetching.md](./data_fetching.md) / [route_handlers.md](./route_handlers.md) / [server_actions.md](./server_actions.md)
