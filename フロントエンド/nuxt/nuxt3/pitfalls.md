# 実務でハマる罠まとめ（Pitfalls）（Nuxt 3）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Nuxt は **SSR（サーバ）と CSR（クライアント）が同じコードを2回走らせる**ため、「ローカルでは動くが本番で壊れる」事故が起きやすい。症状から該当箇所へ素早く飛ぶための索引。

## SSR / ハイドレーション
- **ハイドレーションミスマッチ**：サーバとクライアントでDOMが食い違うと警告＆描画崩れ。`Math.random()` / `Date.now()` / 時刻表示などが原因。クライアント専用描画は `<ClientOnly>` で囲う。→ [rendering.md](./rendering.md)
- **`window` / `document` をサーバで触る**：SSR時は存在せず `ReferenceError`。`onMounted` 内か `import.meta.client` で分岐、またはブラウザAPI依存は `<ClientOnly>` へ。→ [rendering.md](./rendering.md)
- **ブラウザAPI前提のライブラリをトップレベルで読む**：SSRで即死。動的 `import()` を `onMounted` 内で行うか `.client.ts` プラグインに寄せる。→ [rendering.md](./rendering.md)

## 状態 / リクエスト分離
- **`ref` をモジュールスコープで定義して共有**：サーバは1プロセスで複数リクエストを処理するため、**ユーザーAの状態がユーザーBに漏れる**（リクエスト間汚染）。SSR共有状態は必ず `useState` を使う。→ [state.md](./state.md)
- **Pinia ストアをサーバで使い回す**：同様に状態が漏れうる。Nuxtモジュール経由（リクエストごとにストア生成）で使う。→ [state.md](./state.md)
- **`useState` のキー重複**：別物に同じキーを使うと値が混線。キーは一意に。→ [state.md](./state.md)

## データ取得
- **`$fetch` を `<script setup>` 直書きで呼ぶ**：SSRとCSRで**2回叩く（二重取得）**。`useFetch` / `useAsyncData` はサーバの結果をハイドレーションへ引き継ぎ重複排除する。→ [data_fetching.md](./data_fetching.md)
- **`useFetch` / `useAsyncData` を setup の外やイベントハンドラ内で呼ぶ**：これらはコンポーザブルなので**setup直下**で呼ぶのが原則。クリック時の取得は `$fetch` を使う。→ [data_fetching.md](./data_fetching.md)
- **同一データに別キーの `useAsyncData`**：キャッシュが効かず重複取得。キーを揃える。→ [data_fetching.md](./data_fetching.md)
- **`data` を直接書き換える**：`useFetch` の戻り値は ref。再取得は `refresh()` を使う。→ [data_fetching.md](./data_fetching.md)

## 設定 / 秘密情報
- **秘密を `runtimeConfig.public` に入れる＝クライアントへ漏れる**：`public` 配下はブラウザのJSバンドルに載る。APIキー等は `public` でない `runtimeConfig` に置きサーバ側だけで使う。→ [config.md](./config.md)
- **`.env` の値がビルド時に焼き付く誤解**：`runtimeConfig` は実行時に環境変数で上書き可能（`NUXT_` プレフィックス）。デプロイ先で再設定する前提に。→ [config.md](./config.md)
- **`useRuntimeConfig()` をサーバ専用キーでクライアント呼び出し**：`public` 以外はクライアントで空。サーバ（`server/api`）でのみ参照する。→ [config.md](./config.md)

## ルーティング
- **`pages/` ディレクトリが無い**：ファイルベースルーティング自体が無効になり、`app.vue` だけの単一ページに。ルーティングを使うなら `pages/` を作る。→ [pages_routing.md](./pages_routing.md)
- **`pages/` 使用時に `app.vue` で `<NuxtPage />` を置き忘れ**：ページが描画されない。`app.vue` には `<NuxtPage />`（とレイアウト）を置く。→ [pages_routing.md](./pages_routing.md)
- **`<a href>` で内部遷移**：フルリロードが走りSPA遷移にならない。内部リンクは `<NuxtLink>` を使う（プリフェッチも効く）。→ [pages_routing.md](./pages_routing.md)
- **動的ルートのファイル名ミス**：`[id].vue` のような角括弧命名でないとパラメータを拾えない。→ [pages_routing.md](./pages_routing.md)

## 自動インポート / コンポーネント
- **自動インポートが効かない / IDEが赤線**：型生成が古いと未定義扱い。`nuxi prepare`（`.nuxt/` 再生成）で解決。手動 import を足すと二重定義になりがち。→ [components_autoimport.md](./components_autoimport.md)
- **`components/` の命名とディレクトリ**：ネストしたフォルダ名がコンポーネント名の接頭辞になる（`Base/Button.vue` → `<BaseButton>`）。意図しない名前衝突に注意。→ [components_autoimport.md](./components_autoimport.md)
- **`composables/` 直下以外は自動インポート対象外（既定）**：深い階層は拾われない場合がある。配置を確認。→ [components_autoimport.md](./components_autoimport.md)

## レイアウト / ミドルウェア
- **`layouts/` のレイアウトに `<slot />` が無い**：ページ本体が描画されない。レイアウトには必ず `<slot />` を置く。→ [layouts_middleware.md](./layouts_middleware.md)
- **ルートミドルウェアでのリダイレクト無限ループ**：遷移先でも同じ条件が成立すると永久ループ。除外パスを設ける。→ [layouts_middleware.md](./layouts_middleware.md)
- **ミドルウェアでブラウザAPIに依存**：SSR時にも走るため `import.meta.server` / `import.meta.client` で分岐。→ [layouts_middleware.md](./layouts_middleware.md)

## サーバ / Nitro
- **`server/api` で秘密を返してしまう**：レスポンスに含めた瞬間クライアントに渡る。返す項目を絞る。→ [server_routes.md](./server_routes.md)
- **`server/` 内で Vue コンポーザブルを呼ぶ**：`useFetch` 等はコンポーネント文脈用。サーバ側は `$fetch` / `event` を使う。→ [server_routes.md](./server_routes.md)

## SEO / メタ
- **`useHead` / `useSeoMeta` を setup 外で呼ぶ**：反映されない。setup直下で。→ [seo_meta.md](./seo_meta.md)
- **SPAモードでメタが初期HTMLに乗らない**：SEO重視ならSSR/SSGで配信する。→ [rendering.md](./rendering.md)

## 関連
[rendering.md](./rendering.md) / [state.md](./state.md) / [data_fetching.md](./data_fetching.md) / [config.md](./config.md) / [pages_routing.md](./pages_routing.md) / [components_autoimport.md](./components_autoimport.md) / [layouts_middleware.md](./layouts_middleware.md) / [server_routes.md](./server_routes.md) / [seo_meta.md](./seo_meta.md)
