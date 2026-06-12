# 認証・認可（Authentication / Authorization）（Rails 4）

## ひとことで言うと
**認証（Authentication）**は「誰なのか」を確かめること（ログイン）、**認可（Authorization）**は「その人が何をしてよいか」を制御すること（権限）。Rails 4 では `has_secure_password` での自前実装か、`devise`（認証）＋`CanCanCan`/`Pundit`（認可）が定番。

## 役割・なぜ必要か
- ユーザを識別してログイン状態を保ち、本人だけが自分のリソースを操作できるようにするために認証がある。
- ログイン済みでも「管理者だけ削除可」のように操作権限を分けるために認可がある。認証と認可は別の関心事として分けて考える。

## 基本の書き方（コード）

### 自前認証（has_secure_password）
```ruby
# Gemfile: gem "bcrypt"
class User < ActiveRecord::Base
  has_secure_password           # password / password_confirmation と authenticate を提供
  validates :email, presence: true, uniqueness: true
end
```
```ruby
# SessionsController
def create
  user = User.find_by(email: params[:email])
  if user && user.authenticate(params[:password])
    reset_session                       # セッション固定化対策
    session[:user_id] = user.id
    redirect_to root_path, notice: "ログインしました"
  else
    flash.now[:alert] = "メールかパスワードが違います"
    render :new
  end
end

def destroy
  reset_session
  redirect_to root_path
end
```
```ruby
# ApplicationController
private
def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
helper_method :current_user

def require_login
  redirect_to login_path unless current_user
end
```
- マイグレーションに `password_digest:string` カラムが必要。

### devise（認証gem）
```ruby
# Gemfile: gem "devise"
# rails generate devise:install / rails generate devise User
class PostsController < ApplicationController
  before_action :authenticate_user!     # devise が提供
end
```

### 認可（CanCanCan）
```ruby
# app/models/ability.rb
class Ability
  include CanCan::Ability
  def initialize(user)
    user ||= User.new
    can :manage, Post, user_id: user.id   # 自分の投稿だけ全操作
    can :read,   Post                     # 閲覧は誰でも
    can :manage, :all if user.admin?      # 管理者は全部
  end
end
```
```ruby
# コントローラ
def destroy
  @post = Post.find(params[:id])
  authorize! :destroy, @post              # 権限が無ければ例外
  @post.destroy
end
```

## 実務での使い方・定番パターン
- **小規模/学習は has_secure_password**、本格運用は **devise**（確認メール・パスワード再設定・ロック等が揃う）。
- **認可は CanCanCan か Pundit**。`current_user.posts.find(...)` のスコープ縛りも基本の防御。→ [controller.md](./controller.md)
- **ログイン直後に `reset_session`**：セッション固定化攻撃を防ぐ。→ [session_cookie_flash.md](./session_cookie_flash.md)
- パスワードは平文保存しない（`has_secure_password`/devise が bcrypt でハッシュ化）。
- （Rails 4 には `authenticate_by`（7.1）のようなタイミング攻撃対策ヘルパーは無い。`has_secure_password` の `authenticate` を使う。）

## ハマりどころ / アンチパターン
- **`reset_session` 忘れ**：ログイン前後でセッションIDを再生成しないと固定化攻撃に弱い。
- **認証と認可の混同**：ログインできる＝何でもできる、ではない。操作権限は別途チェック。
- **`current_user` を毎回DBアクセス**：`@current_user ||=` でメモ化。
- **認可漏れ**：`Post.find` は他人のも引ける。スコープ（`current_user.posts.find`）か `authorize!` で守る。→ [security.md](./security.md)
- **devise の strong parameters**：カスタム項目を追加したら `configure_permitted_parameters` で許可しないと保存されない。→ [strong_parameters.md](./strong_parameters.md)

## 関連
[controller.md](./controller.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [filters.md](./filters.md) / [security.md](./security.md)
