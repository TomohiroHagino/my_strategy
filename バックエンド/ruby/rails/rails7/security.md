# セキュリティ（Security）（Rails 7）

## ひとことで言うと
Webアプリの定番攻撃（CSRF / mass assignment / XSS / SQLインジェクション等）に対する、**Railsが標準で用意する防御の使い方**。多くは既定で有効だが、「無効化してしまう書き方」を避けるのがポイント。

## 役割・なぜ必要か
- 外部から来る入力（params・ヘッダ・ボディ）は**信用できない**前提で扱う必要がある。
- Railsは安全側のデフォルト（自動エスケープ・CSRFトークン・Strong Parameters）を持つが、`raw` や文字列連結クエリなど**素朴な書き方で穴が開く**。どこが守られ、どこで自分が破りうるかを知るのが目的。

## 基本の書き方（コード）
```ruby
# 1) CSRF: ApplicationController で既定有効
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception   # 7では既定で有効
end
# form_with はトークンを自動埋め込み。手書き<form>やJSのfetchは要対応。

# 2) mass assignment: Strong Parameters で許可キーだけ通す
def user_params
  params.require(:user).permit(:name, :email)   # :admin は通さない
end

# 3) SQLインジェクション: 文字列連結ではなくプレースホルダ
User.where("name = ?", params[:q])          # OK
User.where(name: params[:q])                # OK（ハッシュ形式）
# User.where("name = '#{params[:q]}'")      # NG（絶対に書かない）
```

```erb
<%# 4) XSS: ERBは既定でエスケープ。raw/html_safe を乱用しない %>
<%= @user.name %>                  <%# 自動エスケープされ安全 %>
<%= raw @user.bio %>               <%# NG: 生HTML注入の穴 %>
<%= sanitize @user.bio %>          <%# 許可タグだけ残して無害化 %>
```

## 実務での使い方・定番パターン
- **CSRF**：HTMLフォームは `form_with` を使えば自動でトークンが入る。APIモード（`ActionController::API`）はCookieセッションを使わずトークン認証（Bearer等）に切り替えるのが定石。SPAからJSで叩くなら `csrf_meta_tags` のトークンをヘッダに載せる。
- **認可**：`current_user.posts.find(params[:id])` のように**常にログインユーザのスコープ**で引き、他人のリソース操作を構造的に防ぐ。本格的にはPunditやCanCanCan。→ [controller.md](./controller.md)
- **秘密情報**：APIキー・DBパスワードはコードに書かず `config/credentials.yml.enc`（暗号化）か環境変数 `ENV` で管理。→ [config_credentials.md](./config_credentials.md)
- **静的解析**：`brakeman` をCIに組み込み、SQLインジェクション・mass assignment・安全でないリダイレクト等を自動検出する。
- **HTTPS / Cookie**：本番は `config.force_ssl = true`。セッションCookieは `secure` / `httponly` を有効に。

## ハマりどころ / アンチパターン
- **`raw` / `html_safe` の乱用**：ユーザ入力に付けるとXSS直結。整形済み安全文字列にだけ使う。不安なら `sanitize`。
- **`where` に文字列展開**：`"... #{params[...]}"` は即SQLインジェクション。必ず `?` か名前付き `:key` のプレースホルダ。
- **Strong Parameters の `permit` し忘れ**：そのキーは黙って無視され「保存したのに入らない」。逆に `permit!`（全許可）は危険。→ [strong_parameters.md](./strong_parameters.md)
- **master.key / credentials の鍵をコミット**：`config/master.key` は `.gitignore` 必須。漏れたら鍵を再生成。→ [config_credentials.md](./config_credentials.md)
- **オープンリダイレクト**：`redirect_to params[:url]` は外部誘導の穴。許可リストで検証する。
- **CSRF を安易に `skip_forgery_protection`**：APIで切るなら代替の認証を必ず用意する。

## 関連
[controller.md](./controller.md) / [view.md](./view.md) / [strong_parameters.md](./strong_parameters.md) / [config_credentials.md](./config_credentials.md)
