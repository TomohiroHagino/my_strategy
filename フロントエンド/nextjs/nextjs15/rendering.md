# 描画戦略（SSR / SSG / ISR）（Next.js 15）

## ひとことで言うと
ページを **いつ HTML にするか** の戦略。**静的（SSG, 既定）=ビルド時に作る** / **動的（SSR）=リクエスト毎に作る** / **ISR=静的だが一定間隔で作り直す**、の3つを App Router が用途に応じて自動・宣言で切り替える。

## 役割・なぜ必要か
- **静的（SSG）**：ビルド時に HTML を生成して CDN から配る。**最速・最安・高負荷に強い**。中身が頻繁に変わらないページ（LP・記事・ドキュメント）向け。App Router の**既定**。
- **動的（SSR）**：リクエストごとにサーバで生成。**ユーザー別・最新必須**のページ（ダッシュボード・検索結果）向け。
- **ISR（Incremental Static Regeneration）**：静的の速さを保ちつつ、`revalidate` 秒ごとにバックグラウンドで作り直す。**「だいたい最新でいい」中間**（ニュース一覧・在庫表示）向け。
- これらを理解しないと「速いはずのページが毎回サーバ計算される（＝意図せず動的化）」といった事故になる。

## 基本の書き方（コード）
```tsx
// 1) ISR：このページを 3600 秒ごとに再生成（静的＋定期更新）
export const revalidate = 3600;

export default async function NewsPage() {
  const news = await fetch("https://api.example.com/news").then((r) => r.json());
  return <ul>{news.map((n: any) => <li key={n.id}>{n.title}</li>)}</ul>;
}
```

```tsx
// 2) 動的ルートの静的化：ビルド時に作るパスを列挙（SSG）
//    app/posts/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch("https://api.example.com/posts").then((r) => r.json());
  return posts.map((p: { slug: string }) => ({ slug: p.slug })); // [{slug:"a"},{slug:"b"}]
}

export default async function Post({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params; // Next 15 では params は Promise
  const post = await fetch(`https://api.example.com/posts/${slug}`).then((r) => r.json());
  return <article><h1>{post.title}</h1><p>{post.body}</p></article>;
}
```

```tsx
// 3) 明示的に動的/静的を指定（強制）
export const dynamic = "force-dynamic"; // 常に SSR（毎リクエスト生成）
// export const dynamic = "force-static"; // 常に静的に寄せる
```

```tsx
// 4) ストリーミング：重い部分だけ Suspense で後追い表示
import { Suspense } from "react";

export default function Page() {
  return (
    <main>
      <h1>すぐ出る見出し</h1>
      <Suspense fallback={<p>読み込み中…</p>}>
        <SlowList /> {/* 取得が終わった所からストリームされる */}
      </Suspense>
    </main>
  );
}
```

## 実務での使い方・定番パターン
- **まず静的（既定）で作れないか考える**。動かす必要が出た所だけ動的へ寄せる。
- **`generateStaticParams`** で動的ルート（`[slug]`）をビルド時に量産＝SSG。残りは初回アクセス時生成にもできる。
- **更新頻度に応じて `revalidate`**：秒数で「どれくらい古さを許すか」を宣言（ISR）。`0` 相当の都度取得が要るなら動的化。
- **`export const dynamic`** で意図を固定：常に最新なら `"force-dynamic"`、静的に寄せたいなら `"force-static"`。
- **ストリーミング + `<Suspense>`**：ページ全体を遅い取得に引きずられず、速い所から表示。`loading.tsx` はルート単位の同じ仕組み（→ [server_client_components.md](./server_client_components.md) の構成と合わせる）。
- キャッシュの粒度・タグ再検証は [caching.md](./caching.md) に集約。

## ハマりどころ / アンチパターン
- **意図せず動的化される**：`cookies()` / `headers()` を呼ぶ、または `searchParams` を読むと、そのページは**動的（SSR）扱い**に切り替わる。静的にしたいページで不用意に使わない（使うなら Client 側や下位の Suspense 境界に閉じ込める）。
- **静的/動的の境界の誤解**：1コンポーネントの動的依存が**ページ全体を動的化**することがある。「どこで動的要素を触っているか」を意識して切り分ける。
- **Next 15 の async API 忘れ**：`params` / `searchParams`、`cookies()` / `headers()` は **Promise（要 `await`）** になった。`await` せず分割代入して `undefined` を踏む事故が多い。
- **`generateStaticParams` の戻り値形ミス**：`[{ slug: "x" }]` の配列を返す。`["x"]` のような文字列配列ではない。
- **`revalidate` を付けたのに更新されない**：CDN/ブラウザ側キャッシュや、`fetch` 側の `cache` 指定と矛盾していないか確認（→ [caching.md](./caching.md)）。

## 関連: [caching.md](./caching.md)
