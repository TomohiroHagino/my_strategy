# 描画戦略（SSG / SSR / ISR / CSR の使い分け）（Next.js 12・Pages Router）

## ひとことで言うと
同じページでも「いつHTMLを作るか」を選べる——ビルド時（SSG）／毎リクエスト（SSR）／定期再生成（ISR）／ブラウザ側（CSR）の4択で、**どのデータ取得関数を書くかで描画戦略が決まる**。

## 役割・なぜ必要か
- ページごとに最適な戦略が違う。ほぼ不変のページを毎回サーバで作るのは無駄、認証依存のページをビルド時に固定するのは不可能。**ページ単位で選べる**ことが Pages Router の要点。
- どれを選んでも、サーバ生成HTMLをブラウザが受け取った後にReactが**ハイドレーション**（HTMLにイベントを結びつけ対話可能にする）する。この二段構えを理解しないと「サーバとクライアントの不一致」エラーで詰まる。

## 基本の書き方（コード）

戦略と関数の対応（同じ画面でも書く関数で決まる）。
```
SSG（静的生成・ビルド時）   … getStaticProps（+ 動的なら getStaticPaths）
ISR（定期再生成）          … getStaticProps + revalidate: N
SSR（毎リクエスト）        … getServerSideProps
CSR（ブラウザ側で取得）    … 取得関数なし + useEffect / SWR / React Query
```

```tsx
// SSG: ビルド時に固定。最速・最安。更新はビルドし直し（または ISR）
export const getStaticProps = async () => ({ props: { now: Date.now() } });
// → now はビルド時刻で固定される

// ISR: 上に revalidate を足すだけ。N秒後の最初のアクセスで裏側で再生成
export const getStaticPropsISR = async () => ({
  props: { now: Date.now() },
  revalidate: 60,
});

// SSR: アクセスの度に実行。常に最新だがサーバ負荷とレイテンシは増える
export const getServerSideProps = async () => ({ props: { now: Date.now() } });
// → now はリクエスト時刻
```

```tsx
// CSR: 取得関数を書かず、マウント後にブラウザで取りに行く
import { useEffect, useState } from "react";

export default function ClientOnly() {
  const [data, setData] = useState<any>(null);
  useEffect(() => {
    fetch("/api/feed").then((r) => r.json()).then(setData);
  }, []);
  if (!data) return <p>読み込み中…</p>;
  return <pre>{JSON.stringify(data)}</pre>;
}
```

## 実務での使い方・定番パターン
- **判断フロー**：(1) データはユーザー/リクエストごとに変わる？ → Yes は SSR。(2) 不変に近い？ → SSG。(3) 時々更新でいいが毎回生成は重い？ → ISR。(4) 初期HTMLに要らず操作後に取れば十分？ → CSR。
- **マーケ/ブログ/ドキュメント**：SSG または ISR（高速・SEO・キャッシュ効く）。
- **ダッシュボード/マイページ/認証必須**：SSR（`getServerSideProps` で cookie/トークンを見る）。
- **ハイブリッド**：枠は SSG/SSR で出し、頻繁に変わる部分（通知数・いいね数）だけ CSR（SWR）で後追い更新。
- **ISR のオンデマンド再生成**：`res.revalidate('/path')`（APIルートから）で、編集時だけ即再生成も可能。

## ハマりどころ / アンチパターン
- **App Router の描画モデルと混同**：App Router は Server Components＋`fetch` のキャッシュで戦略を決める（[../nextjs15/](../nextjs15/)）。Pages Router は「どの取得関数を書くか」で決まる。別物。
- **ハイドレーション不一致**：サーバが描いたHTMLとクライアント初回描画が食い違うと警告/崩れ。原因は `Date.now()`/`Math.random()`/`window` 参照/`localStorage` をレンダー本体で使うこと。これらは `useEffect` 内かサーバ側に寄せる。
- **SSG なのに最新を期待**：`getStaticProps` の値はビルド時固定。最新が要るなら ISR か SSR へ。
- **全部 SSR にする**：不変ページまで `getServerSideProps` にするとキャッシュが効かず遅く高コスト。デフォルトは SSG/ISR を検討。
- **CSR だけでSEO重要ページ**：`useEffect` 取得は初期HTMLが空になり、SEO/OGP が弱い。重要ページはサーバ生成（SSG/SSR）に。

## 関連
[data_fetching.md](./data_fetching.md) / [routing.md](./routing.md) / [pitfalls.md](./pitfalls.md)
