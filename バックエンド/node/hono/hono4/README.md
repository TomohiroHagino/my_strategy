# Hono v4 実務リファレンス（索引）

> **この版 = Hono v4（TypeScript前提）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。

## この版のポイント（Honoの肝）
- **マルチランタイム**：同じコードが Cloudflare Workers / Deno / Bun / Node / Vercel / Lambda で動く（アダプタで吸収）。
- **Context `c` が中心**：`c.req`（入力）/ `c.json()`等（出力）/ `c.env`（Workersのバインディング）を `c` 1つで扱う。
- **型安全 RPC**：ルートをチェーンで定義 → `hc<AppType>()` でクライアントが型付きでAPIを呼べる。
- **Web標準（`Request`/`Response`）ベース**：Node固有に縛られない。
- **組込ミドルウェアが豊富**：cors / logger / jwt / cache / secureHeaders / basicAuth …。

## リクエストの流れ（全体像）
```
リクエスト → ミドルウェア（onion: 前処理 → await next() → 後処理）→ ハンドラ
            ハンドラは Context c を受け取り、c.json()/c.text() で返す
```

## 項目（各ファイルへ）

### はじめに / コア
- [getting_started.md](./getting_started.md) … 始め方（create hono / ランタイム / serve）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [routing.md](./routing.md) … ルーティング（app.get / パラメータ / グループ / チェーン）とは
- [context.md](./context.md) … Context `c`（c.req / c.json…）とは ← Honoの中心
- [middleware.md](./middleware.md) … ミドルウェア（onionモデル / 組込）とは

### 検証・エラー・型
- [validation.md](./validation.md) … バリデーション（zod-validator）とは
- [error_handling.md](./error_handling.md) … エラー処理（onError / HTTPException / notFound）とは
- [rpc.md](./rpc.md) … 型安全RPC（hc クライアント）とは ← Honoの目玉

### ランタイム・認証・データ・運用
- [runtimes.md](./runtimes.md) … マルチランタイム（Workers / Deno / Bun / Node）とは ← Honoの目玉
- [auth.md](./auth.md) … 認証（jwt / bearer ミドルウェア）とは
- [database.md](./database.md) … DB（Drizzle / Prisma / D1、内蔵なし）とは
- [testing.md](./testing.md) … テスト（app.request / Vitest）とは
  - [vitest.md](./vitest.md) … Vitest（app.request 組込クライアント・vi.mock・c.env モック）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Hono v4）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
