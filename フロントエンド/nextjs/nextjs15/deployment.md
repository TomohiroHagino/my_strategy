# デプロイ（Vercel / 環境変数）（Next.js 15）

## ひとことで言うと
作ったNext.jsアプリを**本番公開**する手順と、その際に欠かせない**環境変数の扱い**をまとめたもの。最も簡単なのは本家の **Vercel**、自分でサーバを持つなら **セルフホスト（standalone / Docker / Node）**。

## 役割・なぜ必要か
- Next.jsは「ビルド（`next build`）→ 配信」という流れで動く。SSR・Server Actions・Route Handlersがあるため、**静的ファイルを置くだけでは完結しない**（Node実行環境が要る場面が多い）。
- 環境変数は **DB接続先・APIキー** などを環境ごとに差し替えるための仕組み。ここを誤ると**秘密がクライアントに漏れる**、または**本番でだけ落ちる**事故になる。
- 「どの変数がブラウザに出るのか」を理解することが、Next.jsデプロイの最重要ポイント。

## 基本の書き方（コード）
```bash
# 本番ビルド（.next/ を生成）
next build
# ローカルで本番サーバを起動して確認
next start
```
```ts
// 環境変数の使い分け（最重要）
// NEXT_PUBLIC_ 接頭辞 → ビルド時にバンドルへ埋め込まれ「クライアントに公開」される
const apiBase = process.env.NEXT_PUBLIC_API_BASE_URL // ブラウザでも読める

// 接頭辞なし → サーバ（RSC / Server Action / Route Handler）でのみ読める
const dbUrl = process.env.DATABASE_URL // クライアントでは undefined になる
const secret = process.env.STRIPE_SECRET_KEY // 秘密はこちら（NEXT_PUBLIC_ を付けない）
```
```bash
# .env.local（開発用・Git管理外にする）
DATABASE_URL=postgres://localhost:5432/app
STRIPE_SECRET_KEY=sk_test_xxx
NEXT_PUBLIC_API_BASE_URL=http://localhost:3000
```

## 実務での使い方・定番パターン

### Vercel（本家・最も簡単）
```bash
# GitHubリポジトリを Vercel に連携 → push で自動デプロイ
# 環境変数はダッシュボードの Settings > Environment Variables で設定
# （Production / Preview / Development を分けられる）
npm i -g vercel && vercel       # CLIから手動デプロイも可
```
- ビルド・CDN配信・サーバ実行（Functions）・プレビュー環境まで自動。**設定ゼロで動く**のが最大の利点。
- 環境変数は管理画面で注入。`NEXT_PUBLIC_` 付きは**ビルド時に埋め込まれる**ため、値を変えたら**再デプロイ**が必要。

### セルフホスト（standalone / Docker / Node）
```js
// next.config.ts … 依存を含む最小実行ファイルを出力
const nextConfig = { output: 'standalone' }
export default nextConfig
```
```dockerfile
# Dockerfile（multi-stage の要点だけ）
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # → .next/standalone を生成

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]    # standalone が同梱する起動スクリプト
```
```bash
# Node環境で直接動かす場合（PM2 / systemd 等で常駐）
next build && next start -p 3000
```
- `output: 'standalone'` で `node_modules` 込みの最小成果物が出るので、**軽量なDockerイメージ**を作れる。
- ランタイムの環境変数（`DATABASE_URL` 等）は**コンテナ起動時に注入**する（`docker run -e` / k8s Secret / `.env`）。

## ハマりどころ / アンチパターン
- **`NEXT_PUBLIC_` の意味を誤解 → 秘密漏れ**：`NEXT_PUBLIC_` を付けた変数は**ビルド時にJSバンドルへ平文で焼き込まれ、ブラウザから丸見え**になる。APIキー・DBパスワード等の秘密には**絶対に付けない**。漏れたら**即ローテーション**。
- **サーバ変数とクライアント変数の混同**：接頭辞なしの変数を**クライアントコンポーネント**で読むと `undefined`。ブラウザに出したい値だけ `NEXT_PUBLIC_` を付け、秘密はサーバ側（Server Action / Route Handler / RSC）でだけ使う。
- **ビルド時とランタイムの変数の違い**：`NEXT_PUBLIC_` は **`next build` の瞬間に固定**される。Vercelやコンテナで値を変えても、**再ビルドしない限り反映されない**。逆にサーバ専用変数はランタイムに読むので起動時注入でよい。
- **`.env.local` をコミット**：開発用の秘密が入る。必ず `.gitignore`。本番はインフラのENV/管理画面で渡す。
- **`output: 'standalone'` で `public` / `static` のコピー漏れ**：standaloneは `server.js` を出すが、`.next/static` と `public` は**自分でコピー**しないと画像やCSSが404になる（上のDockerfile参照）。
- **`next start` を `next dev` のつもりで本番運用**：`dev` はホットリロード付きの開発専用で重い。本番は必ず `next build` → `next start`（またはstandalone）。

## 関連
[middleware.md](./middleware.md)
