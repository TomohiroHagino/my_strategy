# キャッシュ（Caching）（Next.js 15）

## ひとことで言うと
Next.js が「取得したデータ」や「描画結果」を**どこまで使い回すか**の仕組み。Next 15 で大きく見直され、**`fetch` は既定で都度取得（キャッシュしない）寄り**に変わった。明示的に `force-cache` や `revalidate` を指定してキャッシュする。

## 役割・なぜ必要か
- 同じデータ取得・同じ描画を毎回やり直すのは無駄。**キャッシュで速度とコストを下げる**ためにある。
- ただしキャッシュが効きすぎると**古いデータが出続ける**事故になる。Next 15 は「既定で安全（新鮮）側」に倒し、**速くしたい所だけ明示的にキャッシュする**設計に変えた。
- Next.js のキャッシュは1つではなく **4層**あり、それぞれ役割と無効化方法が違う。混同するとデバッグが沼る。

## 基本の書き方（コード）
```tsx
// 既定：Next 15 では fetch はキャッシュしない（都度取得寄り）
const res = await fetch("https://api.example.com/posts");

// 明示的にキャッシュ（ビルド時/初回取得を使い回す）
const cached = await fetch("https://api.example.com/posts", {
  cache: "force-cache",
});

// 時間ベースで再検証（ISR：60秒ごとに裏で更新）
const revalidated = await fetch("https://api.example.com/posts", {
  next: { revalidate: 60 },
});

// タグを付けておき、後から狙い撃ちで無効化
const tagged = await fetch("https://api.example.com/posts", {
  next: { tags: ["posts"] },
});
```

```tsx
// Server Action / Route Handler 側で無効化（更新後に呼ぶ）
"use server";
import { revalidatePath, revalidateTag } from "next/cache";

export async function createPost(formData: FormData) {
  await savePost(formData);          // DB更新
  revalidateTag("posts");            // tags:["posts"] のキャッシュを破棄
  revalidatePath("/blog");           // /blog のキャッシュを破棄
}
```

## 実務での使い方・定番パターン
4層のキャッシュを役割で理解する：

| 層 | 何をキャッシュ | スコープ / 寿命 | 主な無効化 |
|----|----------------|------------------|------------|
| **Request Memoization** | 同一リクエスト内の同じ `fetch` | 1リクエスト中だけ | 自動（持ち越さない） |
| **Data Cache** | `fetch` で取得したデータ | サーバ側・永続 | `revalidate` / `revalidateTag` |
| **Full Route Cache** | ルートの描画結果（HTML/RSC） | サーバ側・ビルド〜再検証 | `revalidatePath` / 再デプロイ |
| **Router Cache** | クライアントが持つ画面遷移用キャッシュ | ブラウザ・短時間 | `router.refresh()` / 操作 |

- **Request Memoization**：1リクエスト内で同じ `fetch` を複数コンポーネントが呼んでも**1回にまとめる**。重複取得を勝手に防いでくれる（自動）。
- **Data Cache**：`fetch` 結果のサーバ側キャッシュ。`revalidate` で寿命、`tags` で無効化対象を制御する中心。
- **Full Route Cache**：静的に描けるページの描画結果ごとキャッシュ（SSG/ISR の実体）。`revalidatePath` で破棄。
- **Router Cache**：`<Link>` 遷移を速くするためブラウザが持つキャッシュ。更新が画面に反映されない時は `router.refresh()`。
- **更新→無効化はセット**：データを書き換える Server Action では、終わりに `revalidateTag` / `revalidatePath` を必ず呼んで古いキャッシュを捨てる。→ [data_fetching.md](./data_fetching.md)
- **明示的に新鮮にしたい**ページは `export const dynamic = "force-dynamic"` でルートごと毎回描画にもできる。→ [rendering.md](./rendering.md)

## ハマりどころ / アンチパターン
- **14→15 で `fetch` の既定が変わった**：Next 14 までは `fetch` が**既定でキャッシュ（`force-cache`相当）**だったが、15 では**既定でキャッシュしない**に変更。14 の感覚で「キャッシュされているはず」と思い込むとズレる。**速くしたいなら明示的に `cache: "force-cache"` か `revalidate` を付ける**。
- **4種キャッシュの混同**：「データは更新したのに画面が古い」時、どの層が原因かを切り分ける。サーバ側データなら `revalidateTag/Path`、クライアント遷移なら `router.refresh()`。層を取り違えると無効化しても直らない。
- **古いデータが出続ける**：`revalidate` を長くしすぎる／無効化を呼び忘れる、が典型。更新系処理には無効化をセットで書く習慣にする。
- **`revalidatePath` と `revalidateTag` の使い分け**：パス単位で消すか、タグで横断的に消すか。複数ページに跨るデータは **tag** が便利。
- **動的にしたいのに静的化される**：Cookie やヘッダを読まず `fetch` もキャッシュ可能だと、Next がそのルートを静的判定してビルド時に固定する。意図せぬ静的化は `dynamic = "force-dynamic"` 等で明示。

## 関連: [data_fetching.md](./data_fetching.md) / [rendering.md](./rendering.md)
