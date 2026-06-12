# ルーティング（動的ルート / Link / ナビゲーション）（Next.js 15）

## ひとことで言うと
`[id]` のような **動的セグメントでURLの一部を受け取り**、画面間は `<Link>` で移動する——App Router での「URLとデータと遷移」の扱い方。

## 役割・なぜ必要か
- ブログ記事や商品ページのように **件数が可変なページ**は、1ファイル `[id]/page.tsx` で全URLを賄う。`params` でその値を受け取り、対応するデータを引く。
- ページ遷移を `<a>` ではなく **`<Link>`** にすると、Next がプリフェッチ＆クライアント遷移して**全ページ再読み込みを避ける**（速い・状態が保てる）。
- サーバ側では `redirect` / `notFound`、クライアント側では `useRouter` / `usePathname` と、**実行場所ごとに使う道具が分かれている**。これを取り違えないことが要点。

## 基本の書き方（コード）
```tsx
// app/blog/[slug]/page.tsx — 動的ルート。Next 15 では params は Promise なので await
export default async function Post({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params; // 例: /blog/hello → slug = "hello"
  return <h1>記事: {slug}</h1>;
}
```

```tsx
// searchParams（?q=... 等）も Server Component で Promise として受ける
export default async function Search({
  searchParams,
}: {
  searchParams: Promise<{ q?: string }>;
}) {
  const { q } = await searchParams; // /search?q=next → q = "next"
  return <p>検索語: {q ?? "（なし）"}</p>;
}
```

```tsx
// <Link> でのナビゲーション（<a> ではなくこれを使う）
import Link from "next/link";

export default function Nav() {
  return (
    <nav>
      <Link href="/">Home</Link>
      <Link href="/blog/hello">記事へ</Link>
      <Link href={{ pathname: "/search", query: { q: "next" } }}>検索</Link>
    </nav>
  );
}
```

```tsx
// クライアント側の遷移・現在地（"use client" が必須）
"use client";
import { useRouter, usePathname } from "next/navigation";

export function BackButton() {
  const router = useRouter();
  const pathname = usePathname(); // 例: "/blog/hello"
  return <button onClick={() => router.push("/")}>戻る（now: {pathname}）</button>;
}
```

```tsx
// サーバ側の制御フロー：redirect / notFound（Server Component / Server Action 内）
import { redirect, notFound } from "next/navigation";

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const user = await getUser(id);
  if (!user) notFound();          // → not-found.tsx を表示（404）
  if (user.banned) redirect("/"); // → 即リダイレクト
  return <p>{user.name}</p>;
}
```

```tsx
// catch-all: [...slug] は複数セグメントを配列で受ける
// app/docs/[...slug]/page.tsx  →  /docs/a/b/c で slug = ["a","b","c"]
export default async function Docs({
  params,
}: {
  params: Promise<{ slug: string[] }>;
}) {
  const { slug } = await params;
  return <p>{slug.join(" / ")}</p>;
}
```

## 実務での使い方・定番パターン
- **一覧→詳細**：一覧で `<Link href={`/blog/${post.slug}`}>` を並べ、詳細 `[slug]/page.tsx` で `params` から取得→DBアクセス、が王道。
- **`searchParams` でフィルタ/ページング/タブ**を表現すると、URL がそのまま共有・ブックマーク可能な状態になる（クライアントstateに溜めない）。
- **`[...slug]`（catch-all）／`[[...slug]]`（任意・optional catch-all）** はドキュメントやCMSのネストURLに。
- **クライアント遷移後の更新**：フォーム送信後に `router.refresh()` でサーバデータを再取得して反映。Server Actions と組み合わせる。→ [server_actions.md](./server_actions.md)
- **アクティブリンク**は `usePathname()` で現在地と比較して装飾を切り替える。

## ハマりどころ / アンチパターン
- **`<a href>` で内部遷移**：全ページ再読み込みになりプリフェッチも効かない。内部リンクは必ず `<Link>`。外部URLだけ `<a>`。
- **Next 15 で `params` を同期アクセス**：`params.slug` を `await` せず直接読むと型/実行エラー。**`params`・`searchParams` は Promise**。`async` 関数で `await` する（Next 14 以前の同期アクセスとの最大の違い）。
- **`useRouter` / `usePathname` をサーバで使う**：これらは **client 専用**。`"use client"` の無いファイルで使うとエラー。逆に `redirect`/`notFound` は server 側で使う。
- **`next/router`（旧）から import**：App Router では **`next/navigation`** が正。`next/router` は Pages Router 用。
- **`redirect()` を try/catch で握り潰す**：内部的に例外で制御するため、握ると遷移が止まる。catch の外で呼ぶ。
- **`searchParams` 変更でフルリロード**を期待：`<Link>` でのクエリ変更はクライアント遷移。挙動が要らなければ素直に `<Link>` を使う。

## 関連
[app_router.md](./app_router.md) / [layouts.md](./layouts.md)
