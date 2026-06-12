# 設定 / Credentials（Rails 5）

## ひとことで言うと
アプリの**環境ごとの挙動**（DB接続先・ログレベル等）と、**秘密情報**（APIキー・パスワード等）を安全に管理する仕組み。前者は `config/environments/*.rb`、後者は **5.2 では暗号化 credentials**、**5.0/5.1 では `secrets.yml`** や環境変数（ENV）で扱う。

## 役割・なぜ必要か
- 開発・テスト・本番で設定を切り替えたい（例：本番だけキャッシュON、開発だけ詳細ログ）から、`Rails.env` で分岐する仕組みが要る。
- 秘密情報をソースに直書きすると漏洩事故になる。**暗号化して持つ**か**環境変数で外から渡す**ことで、リポジトリに平文の秘密を残さない。

## バージョンによる秘密情報の扱いの違い（重要）
- **Rails 5.0 / 5.1**：`config/secrets.yml`（環境別キーをYAMLで持つ）。**5.1 で `secrets.yml` の暗号化**（`secrets.yml.enc`）も導入された。
- **Rails 5.2**：**暗号化 credentials**（`config/credentials.yml.enc` ＋ `config/master.key`）に移行。`secrets.yml` から置き換わる流れ。

## 環境ごとの設定
```ruby
# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load    = true
  config.log_level     = :info
  config.force_ssl     = true
end
```

## 秘密情報（5.2 の credentials）
```bash
rails credentials:edit
# → config/credentials.yml.enc（暗号化済み）と
#   config/master.key（復号鍵・Git管理外！）が生成される
```
```yaml
# rails credentials:edit で開くYAML（中身は暗号化保存される）
aws:
  access_key_id: AKIA...
  secret_access_key: xxxx
secret_key_base: yyyy
```
```ruby
# 読み出し（5.2）
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.dig(:aws, :secret_access_key)
ENV["DATABASE_URL"]   # 環境変数はこちら
```

## 秘密情報（5.0/5.1 の secrets.yml）
```yaml
# config/secrets.yml
production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
  aws_access_key_id: <%= ENV["AWS_ACCESS_KEY_ID"] %>
```
```ruby
# 読み出し（5.0/5.1）
Rails.application.secrets.aws_access_key_id
```

## 実務での使い方・定番パターン
- **`config/master.key`（5.2）は必ず `.gitignore`**（生成時に自動で入る）。本番へは鍵を `RAILS_MASTER_KEY` 環境変数で渡すのが定番。
- **ENV（環境変数）** はインフラ側で注入。開発時はファイルから読む **dotenv-rails** gem で `.env` を使う（`.env` も Git管理外）。
- **`config_for`** でYAML設定を環境別に読む（`config/settings.yml` を `Rails.application.config_for(:settings)`）。
- 秘密は credentials（5.2）/ secrets.yml（5.0/5.1）/ ENV、純粋な設定値は environments/*.rb や config_for、と住み分ける。

## ハマりどころ / アンチパターン
- **5.0/5.1 で credentials を使おうとする**：`rails credentials:edit` は 5.2 から。それ以前は `secrets.yml`。バージョン取り違えに注意。
- **`master.key` をコミット（重大事故）**：暗号化の意味が消える。誤コミットしたら**鍵と全秘密をローテーション**。→ [security.md](./security.md)
- **本番で鍵/ENV の渡し忘れ**：`RAILS_MASTER_KEY` 未設定だと credentials を復号できず**起動失敗**（5.2）。デプロイ前に確認。
- **`.env` をコミット**：dotenv は開発専用。本番で `.env` 依存にしない。
- **`secret_key_base` を平文で置く**：セッション署名鍵が漏れると改ざんされる。ENV か暗号化で持つ。

## 関連
[security.md](./security.md) / [getting_started.md](./getting_started.md)
