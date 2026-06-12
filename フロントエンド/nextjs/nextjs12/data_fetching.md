# データ取得（getStaticProps / getStaticPaths / getServerSideProps / ISR）（Next.js 12・Pages Router）

## ひとことで言うと
ページファイルから **専用の取得関数を `export`** して、その戻り値の `props` を画面コンポーネントに渡す——ビルド時に取るなら `getStaticProps`、リクエスト毎に取るなら `getServerSideProps`、動的な静的ページのURL一覧は `getStaticPaths` で列挙する。

## 役割・なぜ必要か
- これらの関数は **サーバ（またはビルド時）でだけ動く**ので、秘密鍵・DB接続・サーバ専用APIを安全に呼べる（バンドルに混ざらない）。
- 取得結果をHTMLに織り込んでから返せるので、**初期表示が速く・SEOに強い**。`useEffect` で取りに行く必要がない。
- 「いつ取るか」を関数の種類で宣言する。ビルド時（SSG）／定期再生成（ISR）／毎リクエスト（SSR）を、ページごとに選べる。

## 基本の書き方（コード）

```tsx
// pages/users.tsx … getStaticProps（ビルド時に1回・SSG）
import type { GetStaticProps } from "next";

type User = { id: string; name: string };

export const getStaticProps: GetStaticProps<{ users: User[] }> = async () => {
  const users: User[] = await fetch("https://api.example.com/users").then((r) => r.json());
  return {
    props: { users },
    revalidate: 60, // ← これを付けると ISR：60秒ごとにバックグラウンド再生成
  };
};

export default function Users({ users }: { users: User[] }) {
  return <ul>{users.map((u) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

```tsx
// pages/blog/[slug].tsx … 動的SSG（getStaticPaths でURLを列挙 → 各URLで getStaticProps）
import type { GetStaticPaths, GetStaticProps } from "next";

export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await fetch("https://api.example.com/posts").then((r) => r.json());
  return {
    paths: posts.map((p: { slug: string }) => ({ params: { slug: p.slug } })),
    fallback: false, // 列挙外のURLは404（true / "blocking" で遅延生成も可）
  };
};

export const getStaticProps: GetStaticProps = async ({ params }) => {
  const post = await fetch(`https://api.example.com/posts/${params!.slug}`).then((r) => r.json());
  if (!post) return { notFound: true }; // → 404
  return { props: { post }, revalidate: 300 };
};

export default function Post({ post }: { post: { title: string } }) {
  return <h1>{post.title}</h1>;
}
```

```tsx
// pages/dashboard.tsx … getServerSideProps（リクエスト毎にSSR）
import type { GetServerSideProps } from "next";

export const getServerSideProps: GetServerSideProps = async (context) => {
  // context に req / res / query / params / resolvedUrl
  const token = context.req.cookies.token;
  if (!token) {
    return { redirect: { destination: "/login", permanent: false } };
  }
  const data = await fetch("https://api.example.com/me", {
    headers: { authorization: `Bearer ${token}` },
  }).then((r) => r.json());
  return { props: { data } };
};

export default function Dashboard({ data }: { data: { name: string } }) {
  return <p>ようこそ {data.name}</p>;
}
```

```tsx
// クライアント主導の取得（操作で取り直す・ポーリング）は SWR / React Query
import useSWR from "swr";
const fetcher = (url: string) => fetch(url).then((r) => r.json());

export function Feed() {
  const { data, isLoading } = useSWR("/api/feed", fetcher); // → api_routes.md
  if (isLoading) return <p>読み込み中…</p>;
  return <ul>{data.map((p: any) => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

`getInitialProps`（レガシー・非推奨）。
```tsx
// 最古の方式。SSGの恩恵を消し、サーバ/クライアント両方で走るため扱いづらい。新規では使わない。
Page.getInitialProps = async (ctx) => {
  const data = await fetch("https://api.example.com/x").then((r) => r.json());
  return { data };
};
```

## 実務での使い方・定番パターン
- **使い分け**：毎回最新・認証依存→`getServerSideProps`、ビルド時に固定→`getStaticProps`、定期再生成→`getStaticProps` + `revalidate`（ISR）、クライアント操作で取り直し→SWR/React Query。
- **`fallback` の選び方**：`false`＝列挙外は404（ページ数が確定・少数）、`'blocking'`＝初回アクセスでサーバ生成して待たせる（大量ページ・推奨寄り）、`true`＝先に骨組みを返し `router.isFallback` でローディング表示。
- **`notFound` / `redirect`**：`getStaticProps`/`getServerSideProps` の戻り値で 404 やリダイレクトを宣言できる（手で `res` を書くより安全）。
- **`context.params` を信頼**：動的値はクライアントの `router.query` より、取得関数の `params` で受けるとサーバ側で確定する。

## ハマりどころ / アンチパターン
- **`getServerSideProps`/`getStaticProps` を App Router で使う**：それらは Pages Router 専用。App Router は Server Component 内で `await fetch`（[../nextjs15/](../nextjs15/)）。混同しない。
- **取得関数内でクライアントAPIを参照**：`window`/`localStorage` はサーバで未定義。これらの関数はサーバ/ビルド時実行。
- **`props` に非シリアライズ値**：`Date` や `undefined`、関数はそのまま渡せない（JSON化される）。`Date` は文字列化、`undefined` は `null` か `notFound` に。
- **`getStaticPaths` 無しで `[slug].tsx` を SSG**：動的ルートで `getStaticProps` を使うなら `getStaticPaths` は必須。
- **`getInitialProps` を使い続ける**：SSGの最適化を無効化し、`_app.tsx` に付けると全ページが毎回サーバ実行になる。`getStaticProps`/`getServerSideProps` へ移行。
- **ISR の `revalidate` を秒として誤解**：単位は秒。短すぎる値はオリジン負荷になる。

## 関連
[rendering.md](./rendering.md) / [routing.md](./routing.md) / [api_routes.md](./api_routes.md) / [pitfalls.md](./pitfalls.md)
