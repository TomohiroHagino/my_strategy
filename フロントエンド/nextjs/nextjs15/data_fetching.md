# データ取得（Next.js 15）

## ひとことで言うと
App Router の基本は **「async な Server Component の中で直接 `await fetch()` や DB アクセスをする」**。クライアント側で都度取りたい所だけ TanStack Query などを使う、という二段構えになっている。

## 役割・なぜ必要か
- Server Component は **`async` 関数にできる**ので、`useEffect` + `useState` で取りに行く従来パターンが不要。**サーバ上でデータを取って、その結果込みの HTML を返せる**（速い・SEO に強い・秘密に触れられる）。
- DB クライアントやサーバ専用 API（秘密鍵が要るもの）も Server Component なら**直接呼べる**（クライアントには漏れない → [server_client_components.md](./server_client_components.md)）。
- 一方、**ユーザー操作後に取り直す・無限スクロール・ポーリング**のような「クライアント主導の取得」は Client Component 側で **TanStack Query / SWR** を使うのが定番（キャッシュ・再取得・楽観更新を任せる）。

## 基本の書き方（コード）
```tsx
// app/users/page.tsx … async Server Component で直接 await
export default async function UsersPage() {
  // fetch はサーバ側で実行される。第2引数でキャッシュ/再検証を制御（→ caching.md）
  const res = await fetch("https://api.example.com/users", {
    next: { revalidate: 60 }, // 60秒ごとに再検証（ISR的）。都度取得なら cache: "no-store"
  });
  if (!res.ok) throw new Error("ユーザー取得に失敗");
  const users: { id: string; name: string }[] = await res.json();

  return (
    <ul>
      {users.map((u) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

```tsx
// 並行取得：依存しない取得は Promise.all でまとめて待つ（ウォーターフォール回避）
export default async function Dashboard() {
  const [user, stats] = await Promise.all([
    fetch("https://api.example.com/me").then((r) => r.json()),
    fetch("https://api.example.com/stats").then((r) => r.json()),
  ]);
  return <Profile user={user} stats={stats} />;
}
```

```tsx
// app/feed/Feed.tsx … クライアント主導の取得は TanStack Query
"use client";
import { useQuery } from "@tanstack/react-query";

export function Feed() {
  const { data, isLoading } = useQuery({
    queryKey: ["feed"],
    queryFn: () => fetch("/api/feed").then((r) => r.json()),
  });
  if (isLoading) return <p>読み込み中…</p>;
  return <ul>{data.map((p: any) => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

## 実務での使い方・定番パターン
- **初期表示は Server Component で取る**。一覧・詳細・OGP に効くページは基本これ。`async` + `await` で素直に書く。
- **DB は Server で直接**：Prisma / Drizzle 等をサーバ側で呼び、結果を表示用に整形してから（必要なら）Client へ props で渡す。
- **クライアント主導は TanStack Query / SWR**：操作で再取得・キャッシュ・楽観更新が要る所だけ。Server 取得と役割を分ける。
- **並行化**：依存のない取得は `Promise.all` でまとめる。`await` を直列に並べると待ち時間が積み上がる（ウォーターフォール）。
- **キャッシュ/再検証は `fetch` の第2引数**（`cache` / `next.revalidate`）で宣言。詳細は [caching.md](./caching.md)。
- **部分的に遅延**：重い取得は `<Suspense>` + `loading.tsx` でストリーミングし、先に出せる所から表示（→ [rendering.md](./rendering.md)）。

## ハマりどころ / アンチパターン
- **「Server Component は async にできる」を知らず `useEffect` で取りに行く**：App Router では多くの取得が `await fetch()` 一行で済む。`useEffect` 取得は本当にクライアント主導の時だけ。
- **サーバ取得とクライアント取得の混同**：Server Component の中で TanStack Query を使おうとする（フック=クライアント専用）/ 逆に Client から DB を直接叩こうとする（秘密が漏れる・そもそも動かない）。役割を分ける。
- **ウォーターフォール**：`await a; await b; await c;` と直列に書いて遅い。独立した取得は `Promise.all`。
- **意図せぬキャッシュ挙動**：Next 15 では `fetch` の既定が以前と変わった。古い前提でキャッシュされる/されないを思い込まず、`cache` / `revalidate` を明示する（→ [caching.md](./caching.md)）。
- **クライアントに秘密を渡す**：取得結果の機微フィールドをそのまま Client の props へ流さない（→ [server_client_components.md](./server_client_components.md)）。
- **エラー握り潰し**：`res.ok` を見ずに `res.json()` して落ちる。失敗は明示的に投げ、`error.tsx` で受ける。

## 関連: [server_client_components.md](./server_client_components.md) / [caching.md](./caching.md)
