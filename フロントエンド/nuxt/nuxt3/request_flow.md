# リクエストの流れ・各層は何を返すか（Nuxt 3）

## ひとことで言うと
HTTPリクエストが **server middleware → ルート（`pages/`）→ `useFetch`/`useAsyncData`（`server/api/` を叩く）→ コンポーネント描画 → HTML** と降りる。「どの部分が何を受け取り、何を返すか」を1枚で俯瞰する。フルスタック（Nitroサーバ＋Vue）なので「層」がある。

## 全体の流れ（図）
```
ブラウザ
   │ HTTP リクエスト（GET /users/123 など）
   ▼
[server middleware]   Nitro で先に走る（認証・ヘッダ）   受:event(H3) → 返:なし/throw/redirect
   │ （pass）
   ▼
[ルーティング]    URL → pages/ のファイルを解決
   │
   ▼
[ページ/コンポーネント setup]  描画準備
   │  ▼
   │ [useFetch / useAsyncData]  server/api/ や外部APIを叩く（SSR時はサーバで取得）
   │   │ 受:URL/key → 返:{ data, pending, error }（refで返る）
   │   │           ↘
   │   │            [server/api/x.ts]  Nitroハンドラ  受:event → 返:JSON
   │   ▼
   │ [コンポーネント描画]  data を使ってテンプレートを組む
   │   │ 返:VNode → HTML
   ▼
[HTML 生成 + ハイドレーション]  サーバでHTML、ブラウザで対話化
   ▼
ブラウザ（HTML描画 → クライアントで対話）
```

## 各部分は「何を受け取り・何を返す」か

| 部分 | 受け取る | 返す | 実行場所 |
|---|---|---|---|
| **server middleware** | `event`（H3：リクエスト） | なし / `redirect` / `throw`（通過制御） | サーバ（Nitro） |
| **ページ（pages/）** | route の `params`/`query` | **VNode → HTML** | サーバ→ブラウザ（hydrate） |
| **useFetch / useAsyncData** | URL / key / options | **`{ data, pending, error }`**（`ref`） | SSR時サーバ・以降クライアント |
| **server/api/x.ts** | `event`（method/body/query） | **JSON**（オブジェクトを返すと自動JSON化） | サーバ（Nitro） |
| **コンポーネント** | props / 取得した data | テンプレート（描画） | サーバ→ブラウザ |

- **`useFetch` は SSR と二重取得を防ぐ仕組み**：サーバで取った結果をペイロードに載せ、クライアントは再取得しない。
- **`server/api/` は同居API**：別サーバを立てず Nitro 内でエンドポイントを持てる。

## コードで通して見る
```ts
// 1) server middleware（server/middleware/auth.ts）：全リクエストの前段
export default defineEventHandler((event) => {
  const session = getCookie(event, "session");
  if (!session && event.path.startsWith("/users")) {
    return sendRedirect(event, "/login");   // 通過制御
  }
});

// 2) server/api/users/[id].ts：Nitro ハンドラ → JSON を返す
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, "id");   // 受け＝event
  const user = await db.user.find(id);
  return user;                               // 返り＝JSON（自動シリアライズ）
});
```
```vue
<!-- 3) pages/users/[id].vue：useFetch で取得 → テンプレートで描画 -->
<script setup lang="ts">
const route = useRoute();
const { data: user, pending } = await useFetch(`/api/users/${route.params.id}`);
// useFetch の返り＝{ data, pending, error }（ref）
</script>

<template>
  <p v-if="pending">loading...</p>
  <h1 v-else>{{ user.name }}</h1>   <!-- data を使って描画 → HTML -->
</template>
```

## 実務での使い方・定番パターン
- **取得は `useFetch`/`useAsyncData`**：ページ/コンポーネントの `setup` で呼び、SSRと整合させる。→ [data_fetching.md](./data_fetching.md)
- **APIは `server/api/`**：ファイル＝エンドポイント。`return obj` でJSON化。→ [server_routes.md](./server_routes.md)
- **共有状態は `useState`**：SSR安全な状態（リクエスト跨ぎ汚染なし）。→ [state.md](./state.md)
- **自動インポート**：`components/`・`composables/` は import 不要。→ [components_autoimport.md](./components_autoimport.md)

## ハマりどころ / アンチパターン
- **`useFetch` を `setup` 外/条件分岐内で呼ぶ**：コンポーザブルはトップレベルで。SSR整合が崩れる。
- **`fetch` を直書きして二重取得**：SSRとクライアントで2回走る。`useFetch` を使う。→ [pitfalls.md](./pitfalls.md)
- **サーバ専用処理をコンポーネントに書く**：DB接続等は `server/api/` へ。クライアントに漏れる。
- **`ref` のないグローバル変数で状態共有**：SSRでリクエスト間汚染。`useState` を使う。

## 関連
[layouts_middleware.md](./layouts_middleware.md) / [data_fetching.md](./data_fetching.md) / [server_routes.md](./server_routes.md) / [state.md](./state.md) / [rendering.md](./rendering.md)
