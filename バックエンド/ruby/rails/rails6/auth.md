# 認証・認可（Rails 6）

## ひとことで言うと
認証（authentication）は「誰か」を確認する処理、認可（authorization）は「何ができるか」を制御する処理。認証は has_secure_password / devise、認可は Pundit / CanCanCan が定番。

## 役割・なぜ必要か
- 認証：ログインしているのが誰かを特定する。パスワードは平文保存せず bcrypt でハッシュ化する。
- 認可：ログイン済みでも、自分のリソースしか操作できないように制限する。認可漏れは他人のデータを操作される脆弱性になる。

## 基本の書き方（コード）
```ruby
# Gemfile: gem 'bcrypt'
# users テーブルに password_digest カラムが必要
class User < ApplicationRecord
  has_secure_password
  validates :email, presence: true, uniqueness: true
end

# ログイン（SessionsController）
def create
  user = User.find_by(email: params[:email])
  if user&.authenticate(params[:password])  # bcrypt で照合
    session[:user_id] = user.id
    redirect_to root_path
  else
    flash.now[:alert] = "メールかパスワードが違います"
    render :new
  end
end

def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
```
```ruby
# devise（実務の定番）
# Gemfile: gem 'devise' → rails g devise:install → rails g devise User
# config/routes.rb
devise_for :users
# コントローラ
before_action :authenticate_user!  # 未ログインはログイン画面へ
# current_user が使える
```
```ruby
# 認可：Pundit
class PostPolicy < ApplicationPolicy
  def update?
    record.user_id == user.id
  end
end
# コントローラ
def update
  @post = Post.find(params[:id])
  authorize @post  # PostPolicy#update? が false なら例外
  @post.update!(post_params)
end

# スコープによる認可（最も確実）
@post = current_user.posts.find(params[:id])  # 他人の post は見つからない
```

## 実務での使い方・定番パターン
- 小規模・学習用は `has_secure_password` で十分。本格的な機能（確認メール、パスワードリセット、ロック）は devise。
- 認可は Pundit（Policy クラス）か CanCanCan（Ability クラス）。一覧の絞り込みも Policy Scope で行う。
- リソース取得は `current_user.posts.find` のようにスコープ経由にすると認可漏れを構造的に防げる。

## ハマりどころ / アンチパターン
- `Post.find(params[:id])` は全 post から探すため認可漏れになりやすい。`current_user.posts.find` を使う。
- パスワードを自前でハッシュ化したり平文保存しない。必ず bcrypt（has_secure_password / devise）に任せる。
- CSRF 対策（`protect_from_forgery`）は Rails 既定で有効。フォームの authenticity_token を消さない。
- `authenticate_user!` を入れ忘れたコントローラがあると未ログインでアクセスできてしまう。ApplicationController で既定 ON にし、公開ページだけ `skip_before_action` する。

## 関連
[session_cookie_flash.md](./session_cookie_flash.md) / [security.md](./security.md) / [controller.md](./controller.md) / [filters.md](./filters.md) / [model.md](./model.md) / [config_credentials.md](./config_credentials.md) / [pitfalls.md](./pitfalls.md)
