# メタデータ / SEO（Metadata API）（Next.js 15）

## ひとことで言うと
App Router の **Metadata API**。`page.tsx` / `layout.tsx` から `metadata` をエクスポートするだけで、`<title>` や `<meta>`、OG画像などの **`<head>` タグを Next が生成**してくれる仕組み。SEO・SNSシェアの見栄えをコードで管理する。

## 役割・なぜ必要か
- `<head>` を手で書くと、ページごとにバラつき・抜け漏れが出る。**型付きのオブジェクトで宣言**すれば、title/description/OG が揃う。
- **RSC（React Server Components）でサーバ側レンダ**するため、HTMLに最初からメタ情報・本文が入った状態でクライアントへ届く。クローラがJS実行を待たずに読めるので **SEO に強い**。
- 静的なメタデータは `metadata` オブジェクト、URLパラメータ等で**動的に変える**メタデータは `generateMetadata` 関数、と2つの入口がある。
- `sitemap.ts` / `robots.ts` でサイトマップ・クロール制御も**コードから生成**できる。

## 基本の書き方（コード）
```tsx
// app/layout.tsx … サイト共通の既定メタデータ
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: { default: "My Site", template: "%s | My Site" }, // 子で %s が差し替わる
  description: "実務向け Next.js リファレンス",
  openGraph: {
    title: "My Site",
    description: "実務向け Next.js リファレンス",
    images: ["/og-default.png"], // OG画像（SNSシェア時のサムネ）
    type: "website",
  },
  twitter: { card: "summary_large_image" },
};
```

```tsx
// app/blog/[slug]/page.tsx … 動的メタデータは generateMetadata
import type { Metadata } from "next";

type Props = { params: Promise<{ slug: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;            // Next 15 で params は Promise
  const post = await getPost(slug);         // 記事をDB/API取得
  return {
    title: post.title,                      // template により "記事名 | My Site"
    description: post.excerpt,
    openGraph: { images: [post.ogImage] },  // 記事ごとのOG画像
  };
}

export default async function Page({ params }: Props) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.body}</article>;
}
```

```tsx
// app/sitemap.ts … サイトマップを生成
import type { MetadataRoute } from "next";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getAllPosts();
  return [
    { url: "https://example.com", lastModified: new Date(), priority: 1 },
    ...posts.map((p) => ({
      url: `https://example.com/blog/${p.slug}`,
      lastModified: p.updatedAt,
    })),
  ];
}
```

```tsx
// app/robots.ts … クロール制御
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: "*", allow: "/", disallow: "/admin" },
    sitemap: "https://example.com/sitemap.xml",
  };
}
```

## 実務での使い方・定番パターン
- **`title.template` で統一**：layout に `template: "%s | My Site"` を置けば、各ページは `title: "記事名"` だけで `記事名 | My Site` になる。サイト名の付け忘れが消える。
- **OG画像**：SNS シェア時のサムネは集客に効く。固定画像でも良いが、`opengraph-image.tsx` を置けば**画像を動的生成**（記事タイトル入りのカード等）もできる。
- **動的メタデータは `generateMetadata`**：`[slug]` のような動的ルートは、その記事データを取って title/description/OG を埋める。本文取得と同じデータを使うので、`fetch` のキャッシュ（メモ化）で**二重取得は自動回避**される。→ [caching.md](./caching.md)
- **`sitemap.ts` / `robots.ts`** はファイルを置くだけで `/sitemap.xml` `/robots.txt` が生える。手書き XML が不要。
- **RSC で本文ごとサーバレンダ**＝クローラが内容を読みやすい。これが SEO の土台。描画戦略（SSG/ISR/SSR）の選び方は別項。→ [rendering.md](./rendering.md)

## ハマりどころ / アンチパターン
- **`metadata` は Server Component 専用**：ファイル先頭に `"use client"` があるコンポーネントからは `metadata` / `generateMetadata` を**エクスポートできない**（無視される）。メタデータが効かない時、まず**クライアントコンポーネント化していないか**を疑う。対話部分は子コンポーネントに切り出し、`page.tsx` 自体はサーバのまま保つ。
- **動的なのに静的 `metadata` を使う**：URLやデータで変わるべきタイトルを固定 `metadata` で書くと全ページ同じになる。**変わるなら `generateMetadata`**。
- **Next 15 で `params` は Promise**：`generateMetadata` の `params` は `await` が要る。14 の感覚で同期アクセスすると型エラー/実行時エラー。
- **OG画像のパス・絶対URL**：SNS 側は相対パスを解決できないことがある。`metadataBase` を layout に設定するか、絶対URLで指定する。
- **title 重複・description 未設定**：`template` を使わず各ページでベタ書き → サイト名抜けや重複。description 無しはクローラに不利。共通既定を layout に置いて差分だけ上書きする。

## 関連: [rendering.md](./rendering.md) / [caching.md](./caching.md)
