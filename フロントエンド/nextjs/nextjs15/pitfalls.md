# 実務でハマる罠まとめ（Pitfalls）（Next.js 15）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、エラーの原因切り分けの入口として使う。

## 役割・なぜ必要か
- App Router / RSC は「Reactの延長」に見えて**サーバ/クライアントの境界**や**キャッシュ**で挙動が変わる。Next 15 で既定値も変わったため、症状から該当箇所へ素早く飛ぶ索引が要る。

## サーバ / クライアントコンポーネント
- **`useState` / `useEffect` / `onClick` はServer Componentで使えない**：状態・イベント・ブラウザAPIを使うファイルの先頭に `"use client"` が必要（`You're importing a component that needs useState...` 等のエラー）。→ [server_client_components.md](./server_client_components.md)
- **`"use client"` を付けすぎる**：付けたファイル以下は全部クライアント化。境界は**葉（末端）に寄せ**、データ取得はサーバ側に残す。→ [server_client_components.md](./server_client_components.md)
- **Server → Client に関数やDB接続を渡す**：propsはシリアライズ可能な値のみ。関数・クラスインスタンスは渡せない。→ [server_client_components.md](./server_client_components.md)

## キャッシュ / データ取得（Next 15で要注意）
- **Next 15でfetchキャッシュの既定が変更**：`fetch` は**既定で都度取得（no-store寄り）**に。キャッシュしたいなら明示（`{ cache: 'force-cache' }` / `next: { revalidate }`）。Next 14以前の感覚で「キャッシュされる前提」だとデータが古い/重いの逆事故。→ [caching.md](./caching.md)
- **GET Route Handlerもキャッシュ既定が変更**：Next 15で `route.ts` のGETは既定で**非キャッシュ**。→ [route_handlers.md](./route_handlers.md)

## 描画 / 動的化
- **`cookies()` / `headers()` を呼ぶとそのルートが動的化**：静的化されず毎回サーバ実行になる（ビルド時の事前生成が外れる）。Next 15ではこれらは**async（要 await）**。意図せぬ動的化に注意。→ [rendering.md](./rendering.md)
- **`params` / `searchParams` はPromise（Next 15）**：`page.tsx` の props で `const { id } = await params` のように **await が必須**に。同期アクセスは型エラー/実行時エラー。→ [routing.md](./routing.md)

## ルーティング / ナビゲーション
- **`<a href>` でページ遷移**：フルリロードになりSPA遷移・prefetchが効かない。内部遷移は **`next/link` の `<Link>`** を使う。→ [routing.md](./routing.md)
- **`error.tsx` は必ずクライアントコンポーネント**：先頭に `"use client"` が必要（`reset` 関数を受けるため）。Server Componentで書くと動かない。→ [layouts.md](./layouts.md)

## セキュリティ
- **`NEXT_PUBLIC_` で秘密漏れ**：`NEXT_PUBLIC_` 付き変数は**ビルド時にJSへ焼き込まれブラウザから丸見え**。APIキー・DBパスワードには付けない。漏れたら即ローテーション。→ [deployment.md](./deployment.md)
- **Server Actionの入力検証漏れ**：`"use server"` 関数は**公開エンドポイント同然**。フォームの値を無検証でDBに流すと事故。zod等で**サーバ側バリデーション必須**、認可も自前で。→ [server_actions.md](./server_actions.md)

## メタデータ / SEO
- **`metadata` / `generateMetadata` はServer Componentのみ**：`"use client"` を付けたファイルでは `export const metadata` が無視される。メタはサーバ側のファイルに置く。→ [metadata_seo.md](./metadata_seo.md)

## 関連
[server_client_components.md](./server_client_components.md) / [caching.md](./caching.md) / [rendering.md](./rendering.md) / [routing.md](./routing.md) / [layouts.md](./layouts.md) / [server_actions.md](./server_actions.md) / [deployment.md](./deployment.md) / [metadata_seo.md](./metadata_seo.md)
