# データ取得（useFetch / useAsyncData / $fetch）（Nuxt 3）

## ひとことで言うと
**`useFetch`** = SSR時にサーバ側で取得し、結果をHTMLに埋め込んでクライアントへ渡す（＝二重取得を防ぐ）データ取得コンポーザブル。**`useAsyncData`** はその汎用版、**`$fetch`** は素のHTTPクライアント。

## 役割・なぜ必要か
- SSRでは「**サーバで一度取得 → HTMLに焼き込み → クライアントは再取得せず使い回す（ハイドレーション）**」が理想。素の `fetch` を `onMounted` で呼ぶとサーバでは取らずクライアントだけになり、初回表示が遅く＆ちらつく。
- `useFetch`/`useAsyncData` はこの「サーバで取って `payload` に載せ、クライアントで**重複排除**する」を自動でやってくれる。だからNuxtでのデータ取得は基本これを使う。
- `$fetch` は「**取得そのもの**」を行う低レベルAPI（`ofetch` ベース）。`useFetch` は内部で `$fetch` を使いつつ、SSR連携と状態（`pending`/`error`）管理を足したラッパ、という関係。

## 基本の書き方（コード）
```vue
<script setup lang="ts">
// useFetch : 最も基本。data/pending/error/refresh が返る
const { data: posts, pending, error, refresh } = await useFetch('/api/posts', {
  // lazy: false なら遷移をブロックして取得完了を待つ（既定）
  // server: true なら SSR でも取得（既定）。false でクライアント専用に
})

// useAsyncData : キー＋任意の非同期関数。$fetch を中で呼ぶ典型形
const { data: user } = await useAsyncData('current-user', () =>
  $fetch('/api/me'),
)
</script>

<template>
  <p v-if="pending">読み込み中…</p>
  <p v-else-if="error">取得失敗: {{ error.message }}</p>
  <ul v-else>
    <li v-for="p in posts" :key="p.id">{{ p.title }}</li>
  </ul>
  <button @click="() => refresh()">再取得</button>
</template>
```

```vue
<script setup lang="ts">
// lazy : 取得を待たずに描画を進め、後から埋める（一覧の体感を良くする）
const { data, pending } = await useFetch('/api/feed', { lazy: true })

// クエリ/動的キー : ref を watch して自動再取得。key は重複排除の単位
const page = ref(1)
const { data: list } = await useFetch('/api/items', {
  query: { page },           // page が変わると自動で再取得
  key: 'items-list',         // 明示キー（同キーは結果を共有）
})

// $fetch をイベントハンドラで直接 : POST 等の「副作用」呼び出しはこれでOK
async function createPost() {
  await $fetch('/api/posts', { method: 'POST', body: { title: '新規' } })
  await refreshNuxtData('items-list') // 一覧を再取得
}
</script>
```

## 実務での使い方・定番パターン
- **GET（一覧・詳細表示）= `useFetch`/`useAsyncData`**。SSRに乗せたい・重複排除したい取得は必ずこちら。
- **POST/PUT/DELETE（副作用）= `$fetch` を直接**。ボタン押下などのイベント内では `$fetch` でよい（SSRハイドレーション対象ではないため二重取得問題が起きない）。
- **`refresh()` / `refreshNuxtData(key)`**：作成・更新後に一覧を最新化。`key` 指定で特定データだけ再取得できる。
- **`lazy: true`**：ファーストビューに不要なデータは遅延させ、ページ遷移をブロックしない。`pending` でスケルトン表示。
- **`server: false`**：ユーザー個別・秘匿性の高いデータをSSRに焼き込みたくない時にクライアント専用化。
- **`pick` / `transform`**：`pick: ['id','title']` で不要フィールドを落とし `payload` を軽量化。`transform` で整形してから載せる。
- **サーバAPIと組む**：取得先 `/api/*` は Nitro の `server/api/` で実装するのが定番。→ [server_routes.md](./server_routes.md)

## ハマりどころ / アンチパターン
- **`useFetch` を setup 外で呼ぶ**：`onMounted` やイベントハンドラ内で呼ぶと警告・誤動作。`<script setup>` のトップレベル（コンポーザブルのルート）で呼ぶ。動的取得が必要なら中で `$fetch` を使う。
- **GETを `$fetch` で直に呼んでSSRに載せる**：`<script setup>` 直下で `$fetch('/api/x')` すると**サーバで1回＋クライアントで1回**＝二重取得になる。`useFetch` は同一 `key` で重複排除するのでこれを防げる。表示用GETは `useFetch`/`useAsyncData`。
- **`key` の衝突／欠落**：`useAsyncData` で同じキーを別データに使い回すと結果が混ざる。逆にキーを変えないと別パラメータでも結果が共有されて更新されない。ユニークかつ意味のあるキーを。
- **`await` 忘れ**：`const { data } = useFetch(...)`（await無し）だとSSRで取得完了前に描画され、`data` が `null` のまま焼き込まれる。トップレベル `await` を付ける。
- **大きな `payload`**：取得結果がそのままHTMLに焼かれるので、巨大配列をそのまま `useFetch` するとHTMLが膨らむ。`pick`/`transform`/ページングで削る。
- **`error` を握り潰す**：`error` を表示せず `data` だけ見ると、失敗時に無言で空表示になる。`error` 分岐を必ず書く。

## 関連
[rendering.md](./rendering.md) / [server_routes.md](./server_routes.md) / [state.md](./state.md)
