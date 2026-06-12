# コントローラ（Rails 6）

## ひとことで言うと
ルーティングから渡されたリクエストを受け、モデルを呼び、`render` か `redirect_to` でレスポンスを返す層。Rails 6 では作成・更新の失敗時は単に `render :new` でよい（Turbo前提でないため `status: :unprocessable_entity` は不要）。

## 役割・なぜ必要か
コントローラは「入力（params）の受け取り」「認可」「モデル操作の呼び出し」「レスポンス選択」を担う。ビジネスロジックを詰め込むとFat Controllerになるため、複雑な処理はモデルやサービスへ寄せる（[service_form.md](./service_form.md)）。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!                 # ログイン必須
  before_action :set_post, only: %i[show edit update destroy]

  def index
    @posts = current_user.posts.recent              # 自分のものだけ取得（認可も兼ねる）
  end

  def show; end                                     # @post は before_action で設定済み

  def new
    @post = current_user.posts.new
  end

  def create
    @post = current_user.posts.new(post_params)
    if @post.save
      redirect_to @post, notice: "作成しました"     # 成功はリダイレクト（PRGパターン）
    else
      render :new                                   # 失敗は再描画。Rails6はstatus指定不要
    end
  end

  def update
    if @post.update(post_params)
      redirect_to @post, notice: "更新しました"
    else
      render :edit
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: "削除しました"
  end

  private

  def set_post
    @post = current_user.posts.find(params[:id])    # 他人のidは見つからず404。認可漏れを防ぐ
  end

  def post_params
    params.require(:post).permit(:title, :body)     # Strong Parameters（[strong_parameters.md]）
  end
end
```
```ruby
# JSONのみのAPIは ActionController::API を継承（ビュー/Cookie/CSRF等を省く）
class Api::PostsController < ActionController::API
  def index
    render json: Post.all
  end
end

# 例外を一括処理
rescue_from ActiveRecord::RecordNotFound do
  render plain: "Not Found", status: :not_found
end
```

## 実務での使い方・定番パターン
- 成功時はリダイレクト（PRGでリロード重複POSTを防ぐ）、失敗時は同じフォームを `render`。
- `current_user.posts.find(id)` のように関連経由で取得し、他ユーザーのリソースを自然に弾く（[auth.md](./auth.md)）。
- 共通処理は `before_action`、認証・認可・例外処理は `ApplicationController` か concern に集約（[concern.md](./concern.md) / [filters.md](./filters.md)）。

## ハマりどころ / アンチパターン
- DoubleRenderError：1アクションで `render` と `redirect_to` を両方通すと落ちる。分岐後に `return` するか if/else で片方だけ通す。
- Fat Controller：保存条件分岐や外部API連携をアクションに直書きしない。モデル/サービス/フォームオブジェクトへ。
- Strong Params漏れ：`permit` に無いカラムは保存されず、無言で欠落する。フォーム追加時は `post_params` も更新する。
- 認可忘れ：`Post.find(params[:id])` だと他人のリソースも取れてしまう。スコープを関連経由にする。

## 関連
[routing.md](./routing.md) / [model.md](./model.md) / [view.md](./view.md) / [strong_parameters.md](./strong_parameters.md) / [filters.md](./filters.md) / [auth.md](./auth.md) / [concern.md](./concern.md) / [service_form.md](./service_form.md)
