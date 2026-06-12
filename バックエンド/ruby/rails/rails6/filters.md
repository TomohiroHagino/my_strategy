# フィルタ（Rails 6）

## ひとことで言うと
コントローラのアクションの前後で共通処理を差し込む仕組み。`before_action` で認証や対象レコード取得を、各アクションの実行前にまとめて行う。

## 役割・なぜ必要か
show / edit / update / destroy のそれぞれで `@post = Post.find(params[:id])` を書くのは重複。`before_action :set_post` で一箇所にまとめれば各アクションは本来の処理に集中できる。ログインチェックも同様に共通化する。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_post, only: %i[show edit update destroy]

  def show; end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: "更新しました"
    else
      render :edit
    end
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end
end
```
種類と適用範囲・スキップ。
```ruby
before_action :require_admin, except: %i[index show]
after_action  :log_access
around_action :with_timing
skip_before_action :authenticate_user!, only: %i[index show]  # ログイン不要ページ
```

## 実務での使い方・定番パターン
- `authenticate_user!`（未ログインなら login へ redirect）を `ApplicationController` に置き、全体に効かせる（[auth.md](./auth.md)）。
- `set_xxx` で対象レコードを取得し `only:` で対象アクションを絞る。
- 公開ページだけ `skip_before_action :authenticate_user!, only:` で認証を外す。

## ハマりどころ / アンチパターン
- `before_action` 内で `redirect_to` した後に処理が続くと、アクション本体でも render され「DoubleRenderError」になる。redirect/render したら `return` するか、フィルタ内で止める。
```ruby
def authenticate_user!
  redirect_to login_path and return unless current_user
end
```
- フィルタは `ApplicationController` からの継承順・宣言順で実行される。親で定義した認証を子で意図せず外す/重複させない。
- `skip_before_action` の `only:` 指定がアクション名とズレると、外したいページで効かない / 外したくないページで外れる。

## 関連
[controller.md](./controller.md) / [auth.md](./auth.md) / [strong_parameters.md](./strong_parameters.md) / [session_cookie_flash.md](./session_cookie_flash.md) / [concern.md](./concern.md)
