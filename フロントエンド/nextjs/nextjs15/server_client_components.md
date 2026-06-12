# サーバ/クライアントコンポーネント（Next.js 15）

## ひとことで言うと
App Router では **コンポーネントは既定で「Server Component（RSC）」=サーバ側で実行されて HTML を返す部品**。状態や対話・ブラウザ API が必要な部品だけ、ファイル先頭に **`"use client"`** を書いて「Client Component」へ切り替える。

## 役割・なぜ必要か
- **Server Component（RSC, 既定）**：サーバで実行されるので **DB・秘密鍵・サーバ環境変数へ直接アクセスできる**。レンダリング結果だけをクライアントへ送り、**コンポーネント本体の JS はブラウザへ送らない**（バンドルが軽い・初期表示が速い・SEO に強い）。
- **Client Component（`"use client"`）**：`useState` / `useEffect` / `onClick` などの**対話・状態・ブラウザ API**（`window`・`localStorage` 等）はここでしか使えない。JS がブラウザへ送られ、ハイドレーションされて動く。
- 方針は **「既定はサーバ、対話の必要な葉（leaf）だけクライアント」**。クライアント化を最小限の末端に押し込むほど、送る JS が減って速くなる。

## 基本の書き方（コード）
```tsx
// app/page.tsx … 既定で Server Component（"use client" を書かない）
import { db } from "@/lib/db";
import LikeButton from "./LikeButton";

// Server Component は async にできる（→ data_fetching.md）
export default async function Page() {
  // サーバで実行されるので DB / 秘密に直接触れてよい
  const posts = await db.post.findMany();

  return (
    <main>
      <h1>記事一覧</h1>
      <ul>
        {posts.map((p) => (
          <li key={p.id}>
            {p.title}
            {/* 対話が要る所だけ Client Component を差し込む */}
            <LikeButton postId={p.id} initialCount={p.likes} />
          </li>
        ))}
      </ul>
    </main>
  );
}
```

```tsx
// app/LikeButton.tsx … 対話があるのでクライアント化
"use client";

import { useState } from "react";

// サーバ→クライアントへは「シリアライズ可能な props」だけ渡せる
// （関数・DB接続・Dateインスタンス等の非シリアライズ値は渡せない）
export default function LikeButton({
  postId,
  initialCount,
}: {
  postId: string;
  initialCount: number;
}) {
  const [count, setCount] = useState(initialCount);

  return (
    <button onClick={() => setCount((c) => c + 1)}>
      いいね {count}（{postId}）
    </button>
  );
}
```

## 実務での使い方・定番パターン
- **既定はサーバのまま**置く。`"use client"` は「状態・イベント・ブラウザ API が要る末端の部品」にだけ付ける。
- **サーバで取ってクライアントへ渡す**：データ取得は Server Component で行い、表示・対話用の値を props で Client Component へ流す（上の `initialCount` の形）。
- **Client の中に Server を“子”として差し込める**：`<ClientComp><ServerComp /></ClientComp>` のように **children / props 経由でサーバ部品を渡す**と、対話の殻だけクライアントにして中身はサーバ描画にできる（重い JS を増やさないテク）。
- **`"use client"` 直書きは境界の宣言**：そのファイルと、そこから `import` する配下は**まとめてクライアント扱い**になる（下記）。
- 第三者の対話系ライブラリ（チャート・モーダル等）は Client 側に置く。
- 関連：取得の詳しい話は [data_fetching.md](./data_fetching.md)、土台は [getting_started.md](./getting_started.md)。

## ハマりどころ / アンチパターン
- **Server Component で `useState` / `useEffect` / `onClick` を使うとエラー**（"You're importing a component that needs useState…" 等）。→ そのファイル先頭に **`"use client"`** を付けてクライアント化する。
- **秘密をクライアントに漏らす**：Server で読んだ API キーや DB 結果の機微フィールドを、そのまま Client の props に渡すと**ブラウザに露出**する。クライアントへ渡すのは表示に必要な値だけに絞る。`NEXT_PUBLIC_` 接頭辞の環境変数は**公開前提**である点も注意。
- **`"use client"` の伝播**：`"use client"` を付けたファイルから `import` した部品は、たとえサーバで動かせる内容でも**クライアント側に含まれる**。境界を上位に置きすぎると JS が肥大化する → なるべく**末端**に置く。
- **非シリアライズ値を props で渡す**：関数・クラスインスタンス・DB ハンドル等は Server→Client へ渡せない（更新処理を渡したいなら Server Actions を使う）。
- **`window` / `document` をサーバで参照**：Server Component や描画時に触ると `ReferenceError`。Client Component の `useEffect` 内で使う。

## 関連: [data_fetching.md](./data_fetching.md) / [getting_started.md](./getting_started.md)
