# 認証・認可（Authentication / Authorization）（Rails 5）

## ひとことで言うと
**認証（Authentication）** は「誰であるか」を確かめること（ログイン）、**認可（Authorization）** は「その人が何をしてよいか」を制御すること（権限）。別の関心事なので分けて考える。

## 役割・なぜ必要か
- ユーザを識別し（認証）、識別したユーザに許された操作だけを通す（認可）ことで、なりすましと権限外操作を防ぐ。
- Rails 5 には認証の組み込みフルスタックは無い（`has_secure_password` という土台はある）。実務では **Devise**（認証）＋ **Pundit / CanCanCan**（認可）が定番。

## 認証の基本（自前 / has_secure_password）
```ruby
# モデル: bcrypt gem ＋ password_digest カラムが前提
class User < ApplicationRecord
  has_secure_password                  # password / password_confirmation を提供
  validates :email, presence: true, uniqueness: true
end
```
```ruby
# セッション作成（ログイン）
def create
  user = User.find_by(email: params[:email])
  if user&.authenticate(params[:password])     # bcrypt で照合
    reset_session                              # セッション固定化対策
    session[:user_id] = user.id
    redirect_to root_path, notice: "ログインしました"
  else
    flash.now[:alert] = "メールかパスワードが違います"
    render :new
  end
end

# current_user（ApplicationController に置く）
def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
helper_method :current_user
```

## 認可の基本（Pundit 例）
```ruby
# app/policies/post_policy.rb
class PostPolicy
  attr_reader :user, :post
  def initialize(user, post); @user = user; @post = post; end
  def update?; user.admin? || post.user_id == user.id; end
end
```
```ruby
# コントローラ
def update
  @post = Post.find(params[:id])
  authorize @post                      # update? が false なら例外 → 403
  # ...
end
```

## 実務での使い方・定番パターン
- **Devise**：登録・ログイン・パスワードリセット・確認メール等を一括提供。`before_action :authenticate_user!` / `current_user` が使える。→ [filters.md](./filters.md)
- **認可は Pundit（Policy）か CanCanCan（Ability）**。コントローラに `if` を並べず、ポリシーに集約。
- **スコープで認可を兼ねる**：`current_user.posts.find(id)` なら他人のレコードを引けない（取得時点で認可）。→ [controller.md](./controller.md)
- **APIモード**は session でなく**トークン認証**（JWT / `has_secure_token` / Devise Token Auth）にする。→ [api_mode.md](./api_mode.md)

## ハマりどころ / アンチパターン
- **認証と認可の混同**：ログイン済み＝何でもできる、ではない。ログイン後も操作ごとに認可を確認する。
- **`reset_session` 忘れ**：ログイン直後にセッションを作り直さないと固定化攻撃の余地。→ [session_cookie_flash.md](./session_cookie_flash.md)
- **パスワードを平文/自前ハッシュで保存**：必ず bcrypt（`has_secure_password`）等を使う。→ [security.md](./security.md)
- **認可をビューだけで隠す**：ボタンを非表示にしてもURL直叩きされる。サーバ側（コントローラ/ポリシー）で必ず弾く。
- **`current_user` を `find`（例外版）で書く**：未ログイン時に `RecordNotFound` で落ちる。`find_by`（nil 返し）にする。

## 関連
[filters.md](./filters.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [security.md](./security.md) / [api_mode.md](./api_mode.md)
