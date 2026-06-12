# 周辺インフラ（Rails 共通）

> Rails本体ではないが、実務でRailsとセットでほぼ必ず出てくる**周辺ミドルウェア／インフラ**。
> 版に依存しない（RedisはRedis）ため、版ごとに重複させず**ここに共有**で置き、各版から参照する。
> 版で既定が変わるもの（例: Solid Queue は Rails 8 で既定）は各ファイル内に「どの版で」を明記。

## 一覧
- [redis.md](./redis.md) … Redis（キャッシュ / ジョブ / Action Cable の土台）
- [sidekiq.md](./sidekiq.md) … Sidekiq（Redis前提の定番ジョブ基盤）
- [solid_queue.md](./solid_queue.md) … Solid Queue（DBバックエンド＝Redis不要のジョブ・Rails 8既定）
- [unicorn.md](./unicorn.md) … Unicorn（プロセスフォーク型アプリサーバ）

## 今後足す候補（同じ型で）
- ⬜ solid_cache.md / ⬜ solid_cable.md（Solidトリオの残り）
- ⬜ postgresql.md / ⬜ mysql.md / ⬜ sqlite.md（DBエンジン）
- ⬜ puma.md（アプリサーバ・現行既定）/ ⬜ nginx.md（リバースプロキシ）
- ⬜ elasticsearch.md / ⬜ memcached.md / ⬜ s3.md（Active Storage連携）

## 書式テンプレ
```markdown
# {名前}（Rails の関連事項 / 周辺インフラ）
## ひとことで言うと
## 役割・なぜ必要か（Railsで何に使うか）
## 基本の使い方（コード）
## 実務での勘所
## ハマりどころ
## 関連: [xxx.md](./xxx.md)
```
