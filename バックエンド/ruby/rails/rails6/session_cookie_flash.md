# session / cookie / flash（Rails 6）

## ひとことで言うと
`session` はリクエストをまたいでサーバ側に保持する状態（ログインユーザーIDなど）。`cookies` はブラウザに保存する値。`flash` は次の1リクエストにだけ残るメッセージ。

## 役割・なぜ必要か
HTTP はステートレスなので、ログイン状態を維持するには「誰がログイン中か」を保存する場所が要る。Rails 6 既定の cookie store では session を暗号化して cookie に入れ、`secret_key_base`（credentials）で署名・暗号化する。flash は redirect 後の「保存しました」を一度だけ表示するために使う。

## 基本の書き方（コード）
```ruby
# session：ログイン状態の保持
session[:user_id] = user.id
@user = User.find_by(id: session[:user_id])
reset_session   # ログアウト時はセッション破棄（固定化攻撃対策）

# cookies：ブラウザに保存。signed / encrypted で改ざん・盗み見を防ぐ
cookies[:locale] = "ja"
cookies.signed[:remember_id] = user.id        # 署名付き（改ざん検知）
cookies.encrypted[:token]    = secret_token    # 暗号化（中身も秘匿）
cookies.delete(:locale)
```
flash の使い分け（redirect か render か）。
```ruby
# redirect する場合は flash（次のリクエストに渡る）
redirect_to @post, notice: "作成しました"        # flash[:notice]
redirect_to login_path, alert: "ログインしてください"  # flash[:alert]

# render で同じリクエスト内に表示する場合は flash.now
def create
  @post = Post.new(post_params)
  unless @post.save
    flash.now[:alert] = "保存に失敗しました"
    render :new
  end
end
```
CSRF 保護は既定で有効（`protect_from_forgery` 相当）。`form_with` が自動でトークンを埋める。

## 実務での使い方・定番パターン
- ログイン状態は `session[:user_id]`、ログアウトは `reset_session`。
- redirect 時は `notice:` / `alert:`、render 時は `flash.now[:alert]`。
- 自動ログイン（remember me）は `cookies.encrypted` にトークンを保存。

## ハマりどころ / アンチパターン
- render なのに `flash[:alert]` を使うと、その画面 + 次の画面の二重表示になる。render は必ず `flash.now`。
- session に大きいオブジェクト（モデルそのもの等）を入れると cookie 上限（4KB）を超えて壊れる。ID だけ入れる。
- `secret_key_base` が無いと session を復号できず全ユーザーがログアウト状態になる。本番は credentials / `RAILS_MASTER_KEY` を必ず設定（[config_credentials.md](./config_credentials.md)）。

## 関連
[controller.md](./controller.md) / [filters.md](./filters.md) / [auth.md](./auth.md) / [config_credentials.md](./config_credentials.md) / [security.md](./security.md)
