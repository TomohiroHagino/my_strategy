# 実務でハマる罠まとめ（Pitfalls）（Hono v4）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、症状からの原因切り分けの入口として使う。

## 役割・なぜ必要か
- HonoはWeb標準ベースで素直な一方、**非同期（await）・型安全RPC・マルチランタイム**まわりで定番の事故がある。症状から該当箇所へ素早く飛ぶための索引。

## ミドルウェア / 制御フロー
- **`await next()` 忘れ**：ミドルウェアで `next()` を呼ばない／`await` しないと、後続ハンドラが実行されない・後処理がレスポンス確定後に走る等で詰まる。onionモデルの基本。→ [middleware.md](./middleware.md)

## RPC / 型
- **ルートをチェーンしないと型が出ない**：RPCの型推論はメソッドチェーンで定義したルートからしか効かない。`app.get().post()...` と繋げ、`AppType` をexportする。バラバラ定義だと `hc` 側が `never` 等になる。→ [rpc.md](./rpc.md)

## Context / リクエスト
- **`c.req.json()` は `await` 必須**：ボディ取得は非同期。`const body = await c.req.json()`。付け忘れるとPromiseを掴んで壊れる。`parseBody` 等も同様。→ [context.md](./context.md)
- **`c.env` はランタイム依存**：D1/KV/シークレット等のバインディングは `c.env` 経由。Workersに `process.env` は無い前提。環境ごとに中身が変わる。→ [runtimes.md](./runtimes.md)

## バリデーション
- **検証済みデータは `c.req.valid()` から取る**：validator通過後の値は `c.req.valid('json')` 等で受け取る。生の `c.req.json()` を使うと検証・型付けの恩恵を捨てることになる。→ [validation.md](./validation.md)

## エラー処理
- **`HTTPException` のimport元を間違える**：`import { HTTPException } from 'hono/http-exception'`。場所を誤るとimportできず、意図したステータスで返せない。→ [error_handling.md](./error_handling.md)
- **本番でエラー詳細を返す**：`onError` でスタックトレースや内部メッセージをそのままレスポンスに出すと情報漏えい。本番は汎用メッセージ＋サーバ側ログに。→ [error_handling.md](./error_handling.md)

## ランタイム / 移植性
- **Node固有APIで移植性低下**：`fs` / `process` / Nodeの `Buffer` 直依存などを書くと、Workers/Deno/Bunで動かなくなる。Web標準API（`fetch`/`Request`/`Response`/`crypto`）に寄せる。→ [runtimes.md](./runtimes.md)

## DB
- **DBは内蔵なし・Edge対応を選ぶ**：HonoにDB機能は無い。ORM/ドライバは自前選定で、Edgeで動くもの（Drizzle+D1 / Hyperdrive / serverless接続）を選ぶ。従来型プールはEdgeで枯渇する。→ [database.md](./database.md)

## 関連
[middleware.md](./middleware.md) / [rpc.md](./rpc.md) / [context.md](./context.md) / [runtimes.md](./runtimes.md) / [validation.md](./validation.md) / [error_handling.md](./error_handling.md) / [database.md](./database.md)
