# App Router（app/ ディレクトリ）（Next.js 15）

## ひとことで言うと
`app/` ディレクトリの **フォルダ構成がそのままURL、`page.tsx` がその画面になる** ——という「ファイルベースのルーティング」のしくみ。Next.js 15 の標準ルータ。

## 役割・なぜ必要か
- ルーティング表を手で書かない。**フォルダを掘れば自動でURLができる**ので、構成＝サイトマップになり迷子になりにくい。
- `layout` / `loading` / `error` / `not-found` といった **特殊ファイル**を置くだけで「共通の外枠」「読み込み中」「エラー画面」が差し込まれる。React Suspense / Error Boundary が裏で配線される。
- **React Server Components（RSC）が既定**で、各 `page.tsx` はサーバで実行される。データ取得をコンポーネント内に直接書ける（→ [routing.md](./routing.md)）。

## 基本の書き方（コード）
```text
app/
├─ layout.tsx              # 全体の外枠（"/" 以下すべてに適用）
├─ page.tsx                # "/"            ← トップ画面
├─ about/
│  └─ page.tsx             # "/about"
├─ blog/
│  ├─ layout.tsx           # "/blog" 配下の共通レイアウト（任意）
│  ├─ page.tsx             # "/blog"
│  └─ [slug]/
│     └─ page.tsx          # "/blog/:slug"  ← 動的ルート
└─ (marketing)/            # route group：URLには出ない“まとめ用”フォルダ
   ├─ pricing/page.tsx     # "/pricing"
   └─ contact/page.tsx     # "/contact"
```

```tsx
// app/about/page.tsx —「フォルダ名 = URLセグメント」「page.tsx = 画面」
export default function AboutPage() {
  return <h1>About</h1>; // → /about で表示
}
```

```tsx
// app/blog/layout.tsx — /blog 配下を共通の枠で包む（page.tsx は children に入る）
export default function BlogLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <section>
      <nav>Blog Nav</nav>
      {children}
    </section>
  );
}
```

```text
特殊ファイル（フォルダごとに置ける）
  layout.tsx     … その階層以下の共通外枠（state を保ったまま children だけ差し替わる）
  page.tsx       … そのパスの画面（これが無いとURLにならない）
  loading.tsx    … data取得中に出るフォールバック（Suspense 境界）
  error.tsx      … 例外時の画面（Error Boundary。"use client" 必須）
  not-found.tsx  … 404 / notFound() 呼び出し時の画面
  template.tsx   … layout に似るが、遷移ごとに作り直す版
```

## 実務での使い方・定番パターン
- **URL設計＝フォルダ設計**。`app/dashboard/settings/page.tsx` ＝ `/dashboard/settings`。掘った分だけパスが伸びる、と覚える。
- **route group `(name)`**：丸括弧フォルダは **URLに含まれない**。`(auth)` でログイン系を、`(marketing)` で公開ページをまとめる、など「レイアウトや整理の都合」でグループ化できる。
- **入れ子レイアウト**：各階層に `layout.tsx` を置くと外側→内側に重なる。ヘッダは全体 `layout`、サイドバーは `dashboard/layout` のように責務で分ける。→ [layouts.md](./layouts.md)
- **`loading.tsx` を置くだけ**で、その配下の非同期描画に自動でローディングUIが出る。手で Suspense を書かなくてよい。
- **コロケーション**：`app/blog/_components/Card.tsx` のように `_` 始まりフォルダはルートにならない。画面の近くに部品を置ける。

## ハマりどころ / アンチパターン
- **`page.tsx` が無いとルートにならない**：`app/foo/` を作っただけでは `/foo` は404。画面化には必ず `page.tsx`。最頻出のミス。
- **規約ファイル名は厳守・大小区別**：`Page.tsx` や `index.tsx` ではダメ。`page` / `layout` / `loading` / `error` / `not-found` の名前ぴったりで効く。
- **`layout.tsx` が無限に増殖**：階層ごとに置けるが、共通でないものを上位に置くと全ページに漏れる。責務の境界に置く。
- **`error.tsx` に `"use client"` 忘れ**：Error Boundary はクライアント側機構なので `"use client"` 必須。付け忘れるとビルドエラー。
- **route group をURLの一部と勘違い**：`(marketing)/pricing` のURLは `/pricing`。`/marketing/pricing` ではない。
- **Pages Router の感覚で `pages/` を併設**：両方あると挙動が混ざる。App Router に統一する。→ [getting_started.md](./getting_started.md)

## 関連
[routing.md](./routing.md) / [layouts.md](./layouts.md)
