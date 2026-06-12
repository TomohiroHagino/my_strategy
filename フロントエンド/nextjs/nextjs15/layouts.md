# レイアウト / loading / error（Next.js 15）

## ひとことで言うと
App Router の**特別なファイル名**たち。`layout.tsx` は複数ページで**共有するUIの枠**、`loading.tsx` は読み込み中の表示、`error.tsx` はエラー時の表示。`app/` 配下にこれらを置くだけで、Next がルーティングと自動で結びつけてくれる。

## 役割・なぜ必要か
- **`layout.tsx`**：ヘッダー・サイドバー・フッターなど「ページをまたいで共通の囲い」を1か所で定義。**ネスト**でき、各セグメントの `layout.tsx` が入れ子に適用される。ページ遷移しても**レイアウトは再レンダリングされず状態を保持**する（開いたメニューやスクロール位置が保たれる）。
- **`loading.tsx`**：そのセグメントの読み込み中に出す UI。中身は React の **Suspense** に自動でラップされ、`page.tsx` の `await` が解決するまで表示される。
- **`error.tsx`**：そのセグメントで投げられた例外を捕まえる **error boundary**。アプリ全体を落とさず、その部分だけフォールバック表示できる。

## 基本の書き方（コード）
```tsx
// app/layout.tsx（root layout：アプリに1つ必須。<html><body> はここだけ）
export const metadata = { title: "My App" };

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body>
        <header>共通ヘッダー</header>
        {children}
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/layout.tsx（ネストしたレイアウト。<html> は書かない）
export default function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="dashboard">
      <nav>サイドバー（遷移しても保持される）</nav>
      <main>{children}</main>
    </div>
  );
}
```

```tsx
// app/dashboard/loading.tsx（読み込み中。自動で Suspense にラップされる）
export default function Loading() {
  return <p>読み込み中…</p>;
}
```

```tsx
// app/dashboard/error.tsx（error boundary。"use client" が必須）
"use client";
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void; // セグメントの再描画を試みる
}) {
  return (
    <div role="alert">
      <p>問題が発生しました：{error.message}</p>
      <button onClick={() => reset()}>再試行</button>
    </div>
  );
}
```

## 実務での使い方・定番パターン
- **共通枠は `layout.tsx`**：認証後エリアの `app/(app)/layout.tsx` にナビゲーションを置く、など。レイアウトはサーバコンポーネントのままにし、対話部分だけ子で `"use client"`。
- **`template.tsx`**：`layout.tsx` とほぼ同じだが、**遷移ごとに再マウントされる**（状態を保持しない）。遷移アニメや、ページごとにエフェクトを再実行したい時に使い分ける。
- **`loading.tsx` で体感速度を上げる**：データ取得が重い `page.tsx` の隣に置けば、即座にスケルトンを出せる。細かく出し分けたい所は `page.tsx` 内で `<Suspense>` を手書きする。
- **`error.tsx` は段階的に配置**：セグmeントごとに置くと、壊れた部分だけフォールバックして他は生かせる。最上位の致命的エラーは `app/global-error.tsx`（こちらは `<html><body>` を自前で持つ）。
- レイアウトには**データ取得（async）も書ける**ので、ヘッダーにユーザー名を出す等もここで完結する。

## ハマりどころ / アンチパターン
- **`error.tsx` をサーバコンポーネントのまま書く**：`error.tsx` は `onClick`/`reset` などフックを使うため **`"use client"` が必須**。付け忘れるとビルドエラーになる。
- **root layout を消す／`<html><body>` を書かない**：`app/layout.tsx` は**必須**で、`<html>` と `<body>` を返さないとレンダリングが壊れる。ネストした layout には逆に書かない。
- **layout が状態を保持する前提を忘れる**：遷移してもアンマウントされないので、「ページごとにリセットしたい状態」を layout に置くと残り続ける。リセットしたいなら `template.tsx` か `key` を使う。
- **`error.tsx` は同階層の `layout.tsx` のエラーを捕まえない**：レイアウト自身が投げた例外は**一つ上**の error boundary が捕まえる。レイアウトの失敗に備えるなら親側に置く。
- **`loading.tsx` がページ全体をブロック**：粒度が粗いと「全部待ち」になる。重い部分だけ `<Suspense>` で囲うほうが体感が良い。

## 関連
[app_router.md](./app_router.md)
