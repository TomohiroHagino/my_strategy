# セキュリティ（Security）（Rails 5）

## ひとことで言うと
Webアプリの代表的な攻撃（CSRF / XSS / SQLインジェクション / マスアサインメント等）に対する**Rails 標準の防御と、開発者が守るべき作法**のまとめ。

## 役割・なぜ必要か
- Rails は多くの防御を既定で有効化しているが、書き方を間違えると簡単に穴があく。何が自動で守られ、何を自分で守るかを把握するためにある。

## CSRF（クロスサイトリクエストフォージェリ）
- 通常モードは `ApplicationController` に `protect_from_forgery with: :exception` が既定で入り、`form_with`/`form_for` が自動でトークンを埋める。レイアウトの `<%= csrf_meta_tags %>` が必要。
- **APIモード（`ActionController::API`）には `protect_from_forgery` が無い**。Cookieセッション認証をAPIで使うなら別途トークン対策が要る（トークン認証なら不要）。→ [api_mode.md](./api_mode.md)

## XSS（クロスサイトスクリプティング）
- ERB の `<%= %>` は**自動でHTMLエスケープ**するので既定では安全。
- **`raw` / `html_safe` / `<%== %>` でエスケープを外すと穴**になる。ユーザ入力には使わない。
- HTMLを許可したい場合は `sanitize` ヘルパーでホワイトリスト的に通す。→ [view.md](./view.md)

## SQLインジェクション
```ruby
# NG: 文字列に値を埋め込む
User.where("name = '#{params[:name]}'")
# OK: プレースホルダ / ハッシュ条件
User.where("name = ?", params[:name])
User.where(name: params[:name])
```
- `where(ハッシュ)` か `?` プレースホルダを使う。`order` に生 params を渡すのも危険（カラム名はホワイトリスト化）。

## マスアサインメント
- **Strong Parameters** で許可キーだけ通す。`permit!`（全許可）は使わない。→ [strong_parameters.md](./strong_parameters.md)

## 認証・セッション
- パスワードは bcrypt（`has_secure_password`）等でハッシュ化。平文/自前ハッシュ禁止。
- ログイン直後に `reset_session`（セッション固定化対策）。→ [auth.md](./auth.md) / [session_cookie_flash.md](./session_cookie_flash.md)
- 機密は credentials（5.2）/ secrets.yml（5.0/5.1）/ ENV に。ソース直書き禁止。→ [config_credentials.md](./config_credentials.md)

## Rails 5 のセキュリティ関連トピック
- **5.2：Content Security Policy DSL**（`config/initializers/content_security_policy.rb`）で CSP を宣言的に設定できるようになった。
- **`force_ssl`**（本番でHTTPS強制）を `config.force_ssl = true` で有効化。
- HTTPセキュリティヘッダ（`X-Frame-Options` 等）は既定で一部設定される。

## 実務での使い方・定番パターン
- **`master.key` / `.env` / `secrets.yml`（平文版）を Git に入れない**。誤コミットしたら鍵と秘密を**ローテーション**。
- **強制HTTPS**：本番は `force_ssl`。Cookie に `secure` / `httponly` を付ける。
- **依存の脆弱性チェック**：`bundler-audit` / `brakeman`（静的解析）を CI に組み込む。
- **ファイルアップロード（Active Storage / 5.2）** は拡張子・サイズ・MIMEを自前で検証。→ [active_storage.md](./active_storage.md)

## ハマりどころ / アンチパターン
- **APIモードで CSRF を忘れる**：Cookie認証APIだと穴。トークン認証にするか対策する。
- **`html_safe` の安易な使用**：ユーザ入力を素通しすると XSS。
- **`where` に文字列補間**：SQLi の典型。プレースホルダを使う。
- **認可をビューだけで隠す**：URL直叩きで通る。サーバ側で必ず弾く。→ [auth.md](./auth.md)
- **エラー画面で詳細露出**：本番で例外詳細やスタックトレースを返さない（`config.consider_all_requests_local = false`）。

## 関連
[strong_parameters.md](./strong_parameters.md) / [auth.md](./auth.md) / [config_credentials.md](./config_credentials.md) / [api_mode.md](./api_mode.md)
