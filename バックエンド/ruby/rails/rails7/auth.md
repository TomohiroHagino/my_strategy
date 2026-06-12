# 認証・認可（Authentication / Authorization）（Rails 7）

## ひとことで言うと
- **認証（Authentication）= 「あなたは誰か」** を確かめること（ログイン）。
- **認可（Authorization）= 「その操作をやってよいか」** を判断すること（権限）。
別物。両方そろって初めて「正しいユーザーが、許された操作だけ」できる。

## 役割・なぜ必要か
- 認証だけでは「ログイン済みなら何でもできる」状態になり、**他人のリソースを操作できてしまう**。
- 認可は「このユーザーが、この対象に、この操作を許されているか」を1か所に集約する。
- 認証はライブラリで賄えるが、**認可は業務ルールそのもの**。Policy として明示し、テストできる形に置くのが堅い。

## 基本の書き方（コード）
### 認証：has_secure_password（自前・最小構成）
```ruby
# Gemfile: gem "bcrypt"
# users テーブルに password_digest カラムが必要
class User < ApplicationRecord
  has_secure_password   # password / password_confirmation を扱い、authenticate を生やす
  validates :email, presence: true, uniqueness: true
end

# ログイン（SessionsController）
user = User.find_by(email: params[:email])
if user&.authenticate(params[:password])   # 一致すれば user、不一致なら false
  session[:user_id] = user.id              # セッションに保持 → ログイン状態
end
```
規模が大きい・機能が要るなら **Devise**（登録/ログイン/パスワードリセット/確認メール等を一括提供）。

### 認可：Pundit（Policy オブジェクト）
```ruby
# Gemfile: gem "pundit"
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def update?
    user.admin? || record.user_id == user.id   # 管理者か、自分の投稿だけ更新可
  end
  def destroy? = update?
end

# Controller 側
class PostsController < ApplicationController
  def update
    @post = Post.find(params[:id])
    authorize @post            # PostPolicy#update? を呼ぶ。NG なら Pundit::NotAuthorizedError
    @post.update!(post_params)
  end
end
```

### 認可の基本：current_user 起点でスコープを絞る
```ruby
# そもそも他人のレコードを引かせない（認可の第一歩）
@post = current_user.posts.find(params[:id])   # 自分の投稿しか取得できない
```

## 実務での使い方・定番パターン
- **認証ヘルパー**を `ApplicationController` に置く：`current_user` / `logged_in?` / `require_login`（`before_action`）。
- **`current_user.posts.find`** を徹底＝「URLのIDを差し替えても他人のデータに届かない」。最も効く認可。
- **Pundit**：細かい権限は Policy に集約。`policy_scope(Post)` で一覧の見える範囲も絞る。
- **CanCanCan**：`Ability` クラスに権限を一元定義する流派（Pundit はモデル別 Policy 派）。どちらか一方に統一する。
- ロール管理は `enum role:` や専用テーブルで。`admin?` 等を Policy から参照。

## ハマりどころ / アンチパターン
- **認証だけして認可を忘れる（最頻・最重大）**：ログインさえすれば `Post.find(params[:id])` で他人の投稿を編集・削除できてしまう（IDOR）。必ずスコープ or Policy で絞る。
- **`authorize` の付け忘れ**：Pundit の `verify_authorized`（`after_action`）で「authorize 漏れ」を検知させる。
- **ビューで非表示にしただけ＝認可ではない**：リンクを隠してもエンドポイントは叩ける。サーバ側で必ずチェック。
- **`has_secure_password` で digest カラム名ミス**：既定は `password_digest`。別名なら `has_secure_password :recovery_password`。
- **パスワードを平文ログ出力**：`filter_parameters` に `:password` を入れる（Rails 既定で入っている）。
- **session を信用しすぎる**：権限変更（降格など）が即時反映されない設計に注意。重要操作は都度 DB の権限を確認。

## 関連
[controller.md](./controller.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [model.md](./model.md)
