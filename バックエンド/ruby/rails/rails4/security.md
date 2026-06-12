# セキュリティ（Security）（Rails 4）

## ひとことで言うと
Rails 4 アプリで守るべき定番のセキュリティ対策（**CSRF / XSS / SQLインジェクション / mass assignment / オープンリダイレクト**）と、Rails が標準で用意する防御の使い方。

## 役割・なぜ必要か
- Web アプリは外部入力を扱う以上、攻撃を前提に守る必要がある。Rails は多くの防御を標準で持つが、**書き方を誤ると防御が外れる**ので正しい使い方を押さえる。

## 主要な対策（コード）

### CSRF（クロスサイトリクエストフォージェリ）
```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception   # 4の既定。トークン不一致で例外
end
```
```erb
<%= csrf_meta_tags %>   <%# layout の <head> に。jquery_ujs がAjaxにトークンを付与 %>
```
- `form_for` / `form_tag` は自動でトークンを埋める。生の `<form>` 手書きは `<%= hidden_field_tag :authenticity_token, form_authenticity_token %>` が要る。
- API で外部からPOSTする場合のみ、対象アクションで CSRF を `skip` するか別の認証（トークン）にする。

### XSS（クロスサイトスクリプティング）
```erb
<%= @user.bio %>                 <%# 自動エスケープされる（安全） %>
<%= raw @user.bio %>             <%# エスケープしない（危険・ユーザ入力に使わない） %>
<%= @user.bio.html_safe %>       <%# 同上・危険 %>
<%= sanitize @user.bio %>        <%# 許可タグだけ残す（ユーザHTMLを出すならこれ） %>
```
- `<%= %>` は既定でHTMLエスケープ。`raw` / `html_safe` を**ユーザ入力に付けない**。出す必要があれば `sanitize`。

### SQLインジェクション
```ruby
# NG: 文字列展開（インジェクション）
Post.where("title = '#{params[:q]}'")
# OK: プレースホルダ
Post.where("title = ?", params[:q])
Post.where("title = :q", q: params[:q])   # 名前付き
Post.where(title: params[:q])             # ハッシュ条件（最も安全）
```

### mass assignment
- **Strong Parameters で防ぐ**（4で標準）。`params.require(:user).permit(:email)` のように許可キーを限定。`params.permit!`（全許可）は使わない。→ [strong_parameters.md](./strong_parameters.md)

### オープンリダイレクト
```ruby
# NG: 外部入力をそのまま遷移先に
redirect_to params[:return_to]
# OK: 許可リスト/相対パス限定
redirect_to safe_path(params[:return_to])  # 自前で検証（ホスト無し相対のみ許可 等）
```

## 実務での使い方・定番パターン
- **session 固定化対策**：ログイン直後に `reset_session`。→ [auth.md](./auth.md) / [session_cookie_flash.md](./session_cookie_flash.md)
- **強制HTTPS**：`config.force_ssl = true`（本番）。
- **秘密情報は ENV / secrets.yml**。コードに直書きしない。漏れたらローテート。→ [config_secrets.md](./config_secrets.md)
- **依存の脆弱性チェック**：`bundle-audit`（既知CVE）/ `brakeman`（静的解析）を CI に入れる。Rails 4 は EOL（サポート終了）なので既知脆弱性に特に注意。
- **ファイルアップロード**：paperclip / carrierwave。コンテンツタイプ・拡張子・サイズを検証（Active Storage は無い）。
- パラメータのログ出力フィルタ：`config.filter_parameters += [:password]`。

## ハマりどころ / アンチパターン
- **`raw` / `html_safe` の乱用**：ユーザ入力に付けてXSS。不安なら `sanitize`。
- **`where` に文字列展開**：`"#{params[...]}"` でSQLインジェクション。プレースホルダへ。
- **`params.permit!`**：mass assignment 防御を無効化。使わない。
- **CSRF を全体で `skip`**：APIのつもりで全アクションのCSRFを切ると Web フォームまで無防備に。対象を限定。
- **`reset_session` 忘れ**：固定化攻撃に弱い。
- **Rails 4 が EOL**：本体に新規セキュリティパッチが来ない。可能なら 5 以降へ移行、難しければ脆弱性監視を厚く。

## 関連
[strong_parameters.md](./strong_parameters.md) / [auth.md](./auth.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [config_secrets.md](./config_secrets.md) / [pitfalls.md](./pitfalls.md)
