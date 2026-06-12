# 設定 / secrets.yml / ENV（Rails 4）

## ひとことで言うと
アプリの設定値やAPIキー・DBパスワードなどの秘密情報を管理する仕組み。Rails 4.1〜は **`config/secrets.yml`** が標準で、`Rails.application.secrets` から読む。credentials（暗号化ファイル）は無い。

## 役割・なぜ必要か
- 環境ごと（development / test / production）で異なる設定や、コミットしたくない秘密情報（外部APIキー・`secret_key_base`）を1箇所で管理するためにある。
- 秘密情報はコードに直書きせず、本番値は ENV 経由で注入してリポジトリに含めないのが原則。→ [security.md](./security.md)

## バージョン前提
- **`config/secrets.yml` は Rails 4.1 で導入**。4.0 には無い（4.0 は `config/initializers/secret_token.rb` に `secret_token` を書く形）。
- credentials（`config/credentials.yml.enc` ＋ `master.key`）は **Rails 5.2〜**。Rails 4 には無い。

## 基本の書き方（コード）

### secrets.yml（4.1〜）
```yaml
# config/secrets.yml
development:
  secret_key_base: 開発用の長い文字列

test:
  secret_key_base: テスト用の長い文字列

production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>   # 本番はENVから
  stripe_api_key:  <%= ENV["STRIPE_API_KEY"] %>
```
```ruby
# 読み出し
Rails.application.secrets.secret_key_base
Rails.application.secrets.stripe_api_key
```
- `secret_key_base` は session の暗号化/署名に使う。これが無いと起動しない。→ [session_cookie_flash.md](./session_cookie_flash.md)
- ERB（`<%= ENV[...] %>`）が使えるので、本番値は ENV から差し込む。

### database.yml も ENV 化
```yaml
# config/database.yml
production:
  adapter: postgresql
  database: myapp_production
  username: <%= ENV["DB_USER"] %>
  password: <%= ENV["DB_PASSWORD"] %>
  host:     <%= ENV["DB_HOST"] %>
```

### ENV の管理（開発）
```ruby
# Gemfile（開発用）
gem "dotenv-rails", groups: [:development, :test]
```
```bash
# .env（.gitignore に追加。コミットしない）
SECRET_KEY_BASE=...
STRIPE_API_KEY=...
```

## 実務での使い方・定番パターン
- **`secret_key_base` は ENV から**：`secrets.yml` の production に直書きしない。`rake secret` で生成した値を ENV に置く。
- **開発は dotenv**、本番は **OS/PaaS の環境変数**（Heroku config vars 等）で注入。
- **`secrets.yml` をコミットするなら development/test だけ**。production セクションは ENV 参照のみにして秘密を含めない。
- アプリ独自設定は `config/settings.yml` ＋ `config` gem（settingslogic 等）や `secrets.yml` の任意キーで持つ。
- 環境別の挙動は `config/environments/{development,test,production}.rb`。

## ハマりどころ / アンチパターン
- **`secrets.yml` の production に秘密を直書きしてコミット**：漏洩。ENV 参照に。漏れたら値を再生成・ローテート。→ [security.md](./security.md)
- **`secret_key_base` 未設定で本番起動失敗**：`Missing secret_key_base`。ENV に設定する。
- **4.0 で `secrets.yml` を探す**：無い。`secret_token.rb` を使う（4.1 で `secrets.yml` に移行）。
- **credentials を探す**：Rails 4 には無い（5.2〜）。`secrets.yml` ＋ ENV で代替。
- **`secret_key_base` をうっかり変更**：既存の session/cookie が全て無効化されログアウトされる。
- **ENV 名のタイプミス**：`ENV["..."]` は未定義だと nil。起動時に必須キーの存在チェックを入れると安全。

## 関連
[session_cookie_flash.md](./session_cookie_flash.md) / [security.md](./security.md) / [getting_started.md](./getting_started.md)
