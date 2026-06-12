# セキュリティ（Rails 6）

## ひとことで言うと
Rails 6 は CSRF 保護・出力の自動 HTML エスケープ・パラメータ化クエリ・Strong Parameters・credentials による秘密管理を標準で備える。これらを無効化したり迂回したりしなければ主要な脆弱性は防げる。

## 役割・なぜ必要か
- CSRF / XSS / SQLi / mass assignment は Web アプリで頻出する攻撃で、Rails は既定で対策を有効化している。
- 既定の安全機構を「うっかり外す」コード（文字列連結 SQL、`raw`/`html_safe` 乱用、`permit!`、秘密のハードコード）が実害の主因になる。

## 基本の書き方（コード）

### CSRF
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```
- `form_with`/`form_for` が `authenticity_token` を hidden で埋め込む。`rails-ujs` が非 GET の ajax リクエストに `X-CSRF-Token` ヘッダを自動付与する。
- API モード（`ActionController::API`）はトークン検証を持たないため、別途トークン認証などで保護する。

### XSS（出力エスケープ）
```erb
<%# 既定で HTML エスケープされる（安全） %>
<%= @user.name %>

<%# 危険：エスケープを外す。信頼できる値のみ %>
<%= raw @content %>
<%== @content %>
<%= @content.html_safe %>

<%# ユーザー入力 HTML を出すなら sanitize で許可タグを絞る %>
<%= sanitize @user_html, tags: %w[strong em a], attributes: %w[href] %>
```

### SQL インジェクション
```ruby
# 安全：プレースホルダ / ハッシュ条件
User.where("name = ?", params[:name])
User.where(name: params[:name])

# 危険：文字列連結（絶対禁止）
User.where("name = '#{params[:name]}'")
```
- `order`/`pluck` のカラム名にユーザー入力を渡す場合も許可リストで検証する（カラム名はプレースホルダ化できない）。

### Strong Parameters
```ruby
def user_params
  params.require(:user).permit(:name, :email)
end
# 危険：全許可。mass assignment を防げない
# params.require(:user).permit!
```

### credentials（秘密管理）
```bash
# 暗号化された認証情報を編集する
EDITOR="vim" bin/rails credentials:edit
```
```ruby
Rails.application.credentials.dig(:aws, :secret_access_key)
```
- 復号鍵は `config/master.key`（または `RAILS_MASTER_KEY`）。鍵はリポジトリにコミットしない。

### 通信・解析
```ruby
# config/environments/production.rb
config.force_ssl = true   # HTTPS 強制、Secure cookie
```
```bash
gem install brakeman && brakeman   # 静的解析
bundle audit check --update         # 既知脆弱性 gem 検査
```

## 実務での使い方・定番パターン
- Brakeman を CI のサイクルに組み込み、文字列連結 SQL・`raw`・open redirect 等を機械的に検出する。
- `where` は常にプレースホルダかハッシュ条件で書く。文字列補間を許さないルールにする。
- `html_safe`/`raw` は使用箇所を最小化し、必ずユーザー入力でない値に限定する。ユーザー由来の HTML は `sanitize` で許可タグを絞る。
- `permit` は使うカラムだけを列挙する。`permit!` は禁止。新カラム追加時に permit 追加を忘れないよう変更とセットでレビューする。
- 認可（誰が何をできるか）はコントローラ/ポリシー層で明示的に行う（auth.md）。認証だけでなく所有者チェックを入れる。
- 秘密は credentials か環境変数に置き、ソースに書かない。`bundle audit` で依存 gem の脆弱性を定期確認する。

## ハマりどころ / アンチパターン
- 文字列連結 SQL：`where("... '#{params[...]}' ...")` は SQLi 直結。必ずプレースホルダにする。
- `raw`/`html_safe` 乱用：ユーザー入力をエスケープせず出力すると XSS。表示専用の値でも入力由来なら `sanitize` を通す。
- `permit!` / permit しすぎ：意図しないカラム（`admin` フラグ等）を更新され権限昇格につながる。許可は最小限に。
- CSRF トークン無効化：`skip_forgery_protection` や `protect_from_forgery with: :null_session` を安易に使うと CSRF を受ける。無効化する場合は API トークン等で代替保護する。
- 秘密のハードコード：API キー・パスワードをソースや `config/*.yml` に平文で置く。credentials か ENV に移し、露出した鍵はローテーションする。
- `master.key` のコミット漏れ/紛失：鍵が無いと credentials を復号できず起動時にクラッシュする。鍵は安全に共有し、リポジトリには入れない（config_credentials.md）。

## 関連
[strong_parameters.md](./strong_parameters.md) / [config_credentials.md](./config_credentials.md) / [auth.md](./auth.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [controller.md](./controller.md) / [active_record.md](./active_record.md) / [view.md](./view.md) / [pitfalls.md](./pitfalls.md)
