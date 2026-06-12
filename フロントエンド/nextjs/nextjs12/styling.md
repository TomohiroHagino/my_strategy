# スタイリング（global CSS / CSS Modules / styled-jsx / Sass / Tailwind）（Next.js 12・Pages Router）

## ひとことで言うと
Pages Router では **global CSS は `_app.tsx` でだけ import**、コンポーネント単位のスコープ付きは **CSS Modules（`*.module.css`）**、組込の **styled-jsx**、`*.scss` の **Sass**、ユーティリティの **Tailwind** が使える——用途で使い分ける。

## 役割・なぜ必要か
- スタイルの衝突を防ぐにはスコープが要る。CSS Modules はクラス名を自動でユニーク化し、**他コンポーネントに漏れない**CSSを書ける。
- リセットCSSやフォント・全体トークンのような**本当に全体に効かせたいもの**は global CSS（`_app.tsx` で1回 import）。
- 小さなコンポーネント内で完結させたいなら styled-jsx（Next 組込、追加依存不要）。設計が決まっているなら Tailwind でクラス直書き。

## 基本の書き方（コード）

```tsx
// global CSS は _app.tsx でだけ import 可能
// pages/_app.tsx
import "../styles/globals.css";
import type { AppProps } from "next/app";
export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}
```

```tsx
// CSS Modules（スコープ付き）。ファイル名は *.module.css
// components/Card.module.css:  .card { padding: 16px; border-radius: 8px; }
import styles from "./Card.module.css";

export function Card({ children }: { children: React.ReactNode }) {
  return <div className={styles.card}>{children}</div>;
  // 出力時は .Card_card__a1b2 のようにユニーク化され、他に漏れない
}
```

```tsx
// styled-jsx（Next 組込・追加依存なし）。<style jsx> はそのコンポーネントにスコープ
export function Badge({ label }: { label: string }) {
  return (
    <>
      <span className="badge">{label}</span>
      <style jsx>{`
        .badge {
          background: rebeccapurple;
          color: white;
          padding: 2px 8px;
          border-radius: 999px;
        }
      `}</style>
    </>
  );
}
```

```scss
// Sass：sass を入れて *.module.scss / globals.scss を使う
// $ npm i -D sass
// styles/globals.scss
$brand: #0070f3;
body { color: $brand; }
```

```tsx
// Tailwind：tailwind.config.js の content に pages/ components/ を指定し、globals.css で @tailwind を読む
// styles/globals.css:  @tailwind base; @tailwind components; @tailwind utilities;
export function Button() {
  return <button className="px-4 py-2 rounded bg-blue-600 text-white">送信</button>;
}
```

## 実務での使い方・定番パターン
- **基本は CSS Modules + global の二段**：リセット/フォント/トークンは `globals.css`、各コンポーネントは `*.module.css`。
- **Tailwind 採用なら統一**：プロジェクトで Tailwind を使うなら CSS Modules と混在させず、ユーティリティ中心に揃える（content に `pages/`・`components/` を忘れず指定）。
- **動的クラス**：`className={isActive ? styles.active : styles.idle}` のように条件で切り替え。`clsx`/`classnames` で結合すると読みやすい。
- **フォント**：Next 12 では `next/head` の `<link>` か `_document.tsx` で読み込む（`next/font` は13以降）。
- **CSS変数でトークン**：`:root` に色・余白・タイポを変数化し、各所で参照（ハードコード散乱を防ぐ）。

## ハマりどころ / アンチパターン
- **global CSS をコンポーネントで import**：`_app.tsx` 以外での global CSS import はビルドエラー。スコープが要るなら `*.module.css` に。
- **CSS Modules のファイル名規則違反**：`*.module.css` でないと CSS Modules として扱われずグローバル扱いになる。命名を守る。
- **App Router の `app/globals.css` 流儀と混同**：Pages Router の global は `_app.tsx` で import（[../nextjs15/](../nextjs15/) は `layout.tsx` で import）。
- **Tailwind の content 指定漏れ**：`content` に対象ディレクトリを入れ忘れると、本番ビルドでクラスがパージされ消える。
- **styled-jsx と CSS Modules の乱用混在**：複数方式を1コンポーネントに混ぜると保守しづらい。方針を1つ決める。
- **`:global()` の濫用**：CSS Modules で `:global()` を多用するとスコープの利点が消える。必要最小限に。

## 関連
[app_document.md](./app_document.md) / [getting_started.md](./getting_started.md) / [pitfalls.md](./pitfalls.md)
