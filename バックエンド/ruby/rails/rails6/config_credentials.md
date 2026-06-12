# 設定 / credentials / ENV（Rails 6）

## ひとことで言うと
環境ごとの設定は `config/environments/*.rb`、秘密情報は暗号化された `credentials.yml.enc`、デプロイ先で変わる可変値は環境変数 `ENV`。役割で置き場所を分ける。

## 役割・なぜ必要か
API キーをソースに直書きすると Git に残り漏洩する。Rails 6 では秘密情報を `master.key` で暗号化して `credentials.yml.enc` に入れ、ファイルごと Git にコミットできる（`master.key` だけ Git 管理外）。環境で変わる接続先などは `ENV` で外から渡す。

## 基本の書き方（コード）
```ruby
# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true
  config.x.payment.timeout = 5          # config.x で独自設定を定義
end
```
```ruby
# credentials を編集（$EDITOR で開く。直接 .enc は開けない）
# $ EDITOR="code --wait" rails credentials:edit
# aws:
#   access_key_id: AKIA...
#   secret_access_key: xxxx
Rails.application.credentials.dig(:aws, :access_key_id)
ENV.fetch("DATABASE_URL")               # ENV は fetch で必須化
```
環境別 credentials（本番鍵を分離）。
```bash
# config/credentials/production.yml.enc と production.key を生成
rails credentials:edit --environment production
# 本番では production.key を置くか、環境変数で渡す
RAILS_MASTER_KEY=xxxxx rails server -e production
```

## 実務での使い方・定番パターン
- 秘密情報（APIキー・トークン）は credentials、デプロイ先で変わる値（DB URL・ホスト名）は ENV。
- 本番と開発で鍵を分けたい場合は環境別 credentials を使い、`production.key` は CI / サーバの環境変数 `RAILS_MASTER_KEY` で渡す。
- `master.key` / `production.key` は `.gitignore` 済みであることを必ず確認する。

## ハマりどころ / アンチパターン
- `master.key` が無い環境では credentials を復号できず起動時にクラッシュする。デプロイ時に鍵の受け渡しを忘れない。
- `credentials.yml.enc` をエディタで直接開いても暗号化されていて読めない。必ず `rails credentials:edit` を使う。
- 環境別 credentials を使っているのに本番で `RAILS_MASTER_KEY`（または `production.key`）が未設定だと、本番起動が失敗する。
- 秘密情報を `ENV` ではなく `config.x` や initializer にベタ書きすると Git に残る。

## 関連
[getting_started.md](./getting_started.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [security.md](./security.md) / [multiple_databases.md](./multiple_databases.md) / [console.md](./console.md)
