# 設定 / Credentials（Rails 7）

## ひとことで言うと
アプリの**環境ごとの挙動**（DB接続先・ログレベル等）と、**秘密情報**（APIキー・パスワード等）を安全に管理する仕組み。前者は `config/environments/*.rb`、後者は暗号化された **credentials** や **環境変数（ENV）** で扱う。

## 役割・なぜ必要か
- 開発・テスト・本番で設定を切り替えたい（例：本番だけキャッシュON、開発だけ詳細ログ）から、`Rails.env` で分岐する仕組みが要る。
- 秘密情報をソースに直書きすると漏洩事故になる。**暗号化して持つ**か**環境変数で外から渡す**ことで、リポジトリに平文の秘密を残さない。
- 「コードはGit管理、秘密は別管理」を徹底するための仕組み。

## 基本の書き方（コード）
```ruby
# config/environments/production.rb（環境ごとの設定）
Rails.application.configure do
  config.cache_classes = true
  config.eager_load    = true
  config.log_level     = :info
  config.force_ssl     = true
end
```
```bash
# credentials（暗号化された秘密情報）を編集
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
# 読み出し
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.dig(:aws, :secret_access_key)
ENV["DATABASE_URL"]   # 環境変数はこちら
```

## 実務での使い方・定番パターン
- **環境別 credentials**（Rails 6.1+/7）：環境ごとに鍵とファイルを分ける。
  ```bash
  rails credentials:edit --environment production
  # → config/credentials/production.yml.enc と production.key
  ```
- **`config/master.key` は必ず `.gitignore`**（生成時に自動で入る）。本番へは鍵をデプロイ環境の環境変数 `RAILS_MASTER_KEY` で渡すのが定番。
- **ENV（環境変数）** はインフラ側で注入（Heroku/Render/k8s Secret 等）。開発時はファイルから読む **dotenv-rails** gem で `.env` を使う（`.env` も **Git管理外**）。
- **`config_for`** でYAML設定を環境別に読む（`config/settings.yml` を `Rails.application.config_for(:settings)`）。
- 秘密は credentials か ENV、それ以外の純粋な設定値は environments/*.rb や config_for、と住み分けると整理しやすい。

## ハマりどころ / アンチパターン
- **`master.key` をコミット**（重大事故）。暗号化の意味が消える。誤コミットしたら**鍵と全秘密をローテーション**。→ [security.md](./security.md)
- **本番でENV/鍵の渡し忘れ**：`RAILS_MASTER_KEY` 未設定だと credentials を復号できず**起動失敗**。デプロイ前に必ず確認。
- **`.env` をコミット**：dotenv は開発専用。本番で `.env` 依存にしない（インフラのENVを使う）。
- **`credentials.yml.enc` の競合**：複数人編集でコンフリクトしやすい。環境別ファイルに分割すると軽減。
- 設定値をコードに直書き（**マジック値**）→ environments か ENV へ寄せる。

## 関連
[security.md](./security.md)
