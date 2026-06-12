# デプロイ（build/start・standalone・Vercel・export・環境変数）（Next.js 12・Pages Router）

## ひとことで言うと
本番は `next build` でビルドして `next start` で起動するのが基本——Vercel ならそのまま、Docker なら `output: 'standalone'`、完全静的なら `next export`、設定や秘密は `next.config.js` と `.env` で扱う。

## 役割・なぜ必要か
- 開発の `next dev` は最適化されておらず本番に使えない。本番は **ビルド済みの最適化された成果物**を `next start` で配信する。
- 動かす環境（Vercel / 自前サーバ / Docker / 静的ホスティング）で出力形態が変わる。`getServerSideProps` や API ルートが要るなら Node サーバ、完全に静的なら `export`。
- 環境ごとに変わる値（APIのURL・鍵）は**コードに埋め込まず**環境変数で注入する。

## 基本の書き方（コード）

```bash
# 標準フロー（Node サーバとして動かす：SSR/ISR/API が使える）
npm run build   # = next build
npm run start   # = next start（本番サーバ起動。ポートは PORT 環境変数 or -p）
```

```js
// next.config.js … Docker 向けに最小依存の成果物を出す
/** @type {import('next').NextConfig} */
module.exports = {
  reactStrictMode: true,
  output: "standalone", // .next/standalone に必要最小限を出力（軽量イメージ向き）
};
```

```dockerfile
# standalone 出力を使う Dockerfile の要点
# ... ビルドステージで next build 実行後 ...
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]   # standalone は server.js を生成する（next start 不要）
```

```bash
# 完全静的化（Next 12 の next export）。SSR/API/ISR は使えない
next build && next export   # → out/ に静的HTMLを出力（静的ホスティングに置ける）
```

環境変数。
```bash
# .env.local（gitignore する。秘密はここ）
DATABASE_URL=postgres://...           # サーバ専用（クライアントに出ない）
NEXT_PUBLIC_API_BASE=https://api.example.com  # NEXT_PUBLIC_ 接頭辞はブラウザにも露出
```

```ts
// 参照
const db = process.env.DATABASE_URL;            // サーバ側（getServerSideProps/API/middleware）
const api = process.env.NEXT_PUBLIC_API_BASE;   // クライアントでも読める（公開前提の値のみ）
```

## 実務での使い方・定番パターン
- **Vercel**：リポジトリ連携で `build` 自動実行。`getServerSideProps`/ISR/API/ミドルウェアがそのまま動く。環境変数はダッシュボードで設定。最も手間が少ない。
- **自前 Node サーバ**：`next build` → `next start`。プロセス管理は pm2 等、前段に nginx を置く構成が定番。
- **Docker**：`output: 'standalone'` で軽量イメージ。`node server.js` で起動（`next start` 不要）。
- **静的ホスティング（S3/CDN等）**：SSR/API が不要なサイトだけ `next export`。動的機能が要るなら不可。
- **環境変数の境界**：ブラウザに見せてよい値だけ `NEXT_PUBLIC_` を付ける。秘密は接頭辞なし＝サーバ専用。

## ハマりどころ / アンチパターン
- **`next export` で SSR/API を期待**：`export`（Next 12）は完全静的化。`getServerSideProps`・API ルート・ISR・ミドルウェアは動かない。これらが要るなら `next start`/Vercel/standalone。
- **秘密に `NEXT_PUBLIC_` を付ける**：その値はクライアントバンドルに焼き込まれ漏れる。秘密は接頭辞なし。
- **環境変数のビルド時固定**：`NEXT_PUBLIC_*` はビルド時に埋め込まれる。実行時に変えたい値はサーバ側参照（接頭辞なし）で扱う。
- **`next dev` で本番運用**：未最適化で遅く不安定。本番は必ず `build` + `start`。
- **`.env.local` をコミット**：秘密が漏れる。`.gitignore` し、`.env.example` で項目だけ共有。
- **App Router 向けの設定を流用**：`output`/`env` の基本は共通だが、App Router 固有のキャッシュ・RSC設定は Pages Router に無い（[../nextjs15/](../nextjs15/)）。

## 関連
[getting_started.md](./getting_started.md) / [middleware.md](./middleware.md) / [api_routes.md](./api_routes.md) / [pitfalls.md](./pitfalls.md)
