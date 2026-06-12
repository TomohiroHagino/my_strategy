# リクエストの流れ・各層は何を返すか（Next.js 12・Pages Router）

## ひとことで言うと
HTTPリクエストが **middleware → `pages/xxx` の解決 → `getServerSideProps`/`getStaticProps`（サーバでデータ取得）→ ページ関数（props）→ HTML** と降りる。「どの部分が何を受け取り、何を返すか」を1枚で俯瞰する。フルスタック（サーバ実行あり）なので「層」がある。

## 全体の流れ（図）
```
ブラウザ
   │ HTTP リクエスト（GET /users/123 など）
   ▼
[middleware.ts]   Edge で先に走る（認証・リダイレクト）   受:NextRequest → 返:NextResponse
   │ （pass）
   ▼
[ルーティング]    URL → pages/ のファイルを解決
   │
   ├─ ページの場合 ───────────────────────────────┐
   │   ▼                                          │
   │ [getServerSideProps / getStaticProps]  サーバでデータ取得
   │   │ 受:context(params/query) → 返:{ props }
   │   ▼
   │ [ページ関数 Xxx(props)]   props を受けて JSX を返す
   │   │ 受:props → 返:JSX
   │   ▼
   │ [HTML 生成 + ハイドレーション]  サーバでHTML、ブラウザで対話化
   │
   └─ API の場合 ─────────────────────────────────┐
       ▼                                          │
     [API Route (pages/api/x.ts)]  handler(req,res)  受:req → 返:res.json()
   ▼
ブラウザ（HTML描画 or JSON受信）
```

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 実行場所 |
|---|---|---|---|
| **middleware.ts** | `NextRequest`（URL/cookie） | `NextResponse`（通過 / redirect / rewrite） | Edge |
| **getServerSideProps** | `context`（`params`/`query`/`req`） | **`{ props }`**（毎リクエスト・SSR） | サーバ（リクエスト毎） |
| **getStaticProps** | `context`（`params`） | **`{ props }`**（ビルド時・SSG/ISR） | サーバ（ビルド時） |
| **ページ関数** | `props` | **JSX**（→HTML化） | サーバ→ブラウザ（hydrate） |
| **API Route** | `req`（`method`/`body`/`query`） | **`res`**（`res.json()`/`res.status()`） | サーバ |

- **コンポーネントは全部クライアント扱い**（React Server Components ではない）。サーバ実行はデータ取得関数とAPI Routeだけ。
- **`getServerSideProps` は毎リクエスト、`getStaticProps` はビルド時**（＋ISRで再生成）に走る。

## コードで通して見る
```ts
// 1) middleware.ts：Edge で前段
import { NextResponse, type NextRequest } from "next/server";
export function middleware(req: NextRequest) {
  if (!req.cookies.get("session")) {
    return NextResponse.redirect(new URL("/login", req.url)); // 返り＝NextResponse
  }
  return NextResponse.next();
}
export const config = { matcher: ["/users/:path*"] };

// 2) pages/users/[id].tsx：getServerSideProps でサーバ取得 → props を返す
import type { GetServerSideProps } from "next";

export const getServerSideProps: GetServerSideProps = async (ctx) => {
  const res = await fetch(`https://api.example.com/users/${ctx.params!.id}`);
  const user = await res.json();
  return { props: { user } };             // 返り＝{ props }（ページへ渡る）
};

// 3) ページ関数：props を受け取り JSX を返す
export default function UserPage({ user }: { user: { name: string } }) {
  return <h1>{user.name}</h1>;            // 返り＝JSX（HTML化 → hydrate）
}

// 4) pages/api/users.ts：API ハンドラ → res で返す
import type { NextApiRequest, NextApiResponse } from "next";
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== "POST") return res.status(405).end(); // 受け＝req
  const created = await db.user.create({ data: req.body });
  res.status(201).json(created);          // 返り＝res.json()
}
```

## 実務での使い方・定番パターン
- **取得タイミングで関数を選ぶ**：毎回最新が要る→`getServerSideProps`、固定/低頻度更新→`getStaticProps`（＋`getStaticPaths`/ISR）。→ [data_fetching.md](./data_fetching.md)
- **API は `pages/api`**：`req.method` で分岐し `res.json()` で返す。→ [api_routes.md](./api_routes.md)
- **共通枠は `_app.tsx`/`_document.tsx`**：Provider・global CSS・html構造。→ [app_document.md](./app_document.md)
- **App Router との違いを把握**：`page.tsx`/Server Components ではなく `pages/`＋props 方式。→ [vs_app_router.md](./vs_app_router.md)

## ハマりどころ / アンチパターン
- **App Router の作法を混ぜる**：`"use server"`/`route.ts` は無い。Pages では取得関数＋API Route。
- **`getServerSideProps` の props に非シリアライズ値**：Date等はそのまま渡せない。文字列化して渡す。
- **`window is not defined`**：取得関数・初期描画はサーバ実行。`window` 参照は `useEffect` 内へ。→ [pitfalls.md](./pitfalls.md)
- **API Route で `res` を返し忘れる**：レスポンス未終了でハング。必ず `res.json()`/`res.end()`。

## 関連
[middleware.md](./middleware.md) / [data_fetching.md](./data_fetching.md) / [api_routes.md](./api_routes.md) / [app_document.md](./app_document.md) / [vs_app_router.md](./vs_app_router.md)
