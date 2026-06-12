# コントローラ（Controller）（Rails 5）

## ひとことで言うと
リクエストを受け取り、**モデルを呼んでデータを用意し、ビュー（HTML）か JSON を返す「司令塔」**。`ApplicationController` を継承し、`app/controllers/` に置く。APIモードでは `ActionController::API` を継承する。

## 役割・なぜ必要か
- ルーティングから渡されたリクエストの「交通整理」をする層。入力（`params`）の受け取り、認証/認可、モデル操作の呼び出し、応答（render/redirect）の決定を担う。
- ここに業務ロジックを書きすぎると太る（Fat Controller）。**判断や計算はモデル/Serviceへ寄せ、コントローラは薄く**保つのが原則。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_post, only: %i[show edit update destroy]

  def index
    @posts = current_user.posts.recent   # モデルに任せる
  end

  def create
    @post = current_user.posts.build(post_params)
    if @post.save
      redirect_to @post, notice: "作成しました"
    else
      render :new                        # Rails 5 は status 指定不要（Turbo が無いため）
    end
  end

  private

  def set_post
    @post = current_user.posts.find(params[:id])   # current_user 経由で認可も兼ねる
  end

  def post_params
    params.require(:post).permit(:title, :body)    # Strong Parameters
  end
end
```

## APIモードでのレスポンス（Rails 5）
```ruby
class Api::V1::PostsController < ApplicationController  # 全体が --api なら ActionController::API 継承
  def show
    post = Post.find(params[:id])
    render json: post                                   # JSON を返す
  end

  def create
    post = Post.new(post_params)
    if post.save
      render json: post, status: :created               # 201
    else
      render json: { errors: post.errors }, status: :unprocessable_entity  # 422
    end
  end
end
```
- HTML応答（`render :new`）と異なり、API では **明示的に `render json:` ＋ HTTPステータス**を返す。
- ビュー無しのレスポンス整形は `jbuilder`（ビューテンプレート）や ActiveModelSerializers が定番。→ [api_mode.md](./api_mode.md)

## 実務での使い方・定番パターン
- **before_action** で共通処理（認証・対象レコード取得）を切り出す。→ [filters.md](./filters.md)
- **Strong Parameters**（`post_params`）で許可キーだけ通す。→ [strong_parameters.md](./strong_parameters.md)
- **`current_user.posts.find(...)`** のように常にログインユーザのスコープで引くと、他人のリソース操作を自然に防げる（認可の基本）。
- **`respond_to`** でHTML/JSON両対応にできる（`format.html` / `format.json`）。SJR（`*.js.erb`）を返す `format.js` も Rails 5 ではまだ使う。→ [javascript.md](./javascript.md)
- **`rescue_from`** で例外を共通ハンドリング（`RecordNotFound` → 404 等）。

## ハマりどころ / アンチパターン
- **二重 render/redirect**（`DoubleRenderError`）：1アクションで応答は1回。分岐後に `return` を忘れない。
- **Fat Controller**：分岐・計算が増えたらモデル/Serviceへ。
- **Strong Parameters 漏れ**：permit し忘れたキーは黙って無視される＝「保存したのに入らない」。
- **APIモードで CSRF 例外に注意**：`ActionController::API` には `protect_from_forgery` が無い。Cookieセッション認証をAPIで使うなら別途トークン対策が要る（トークン認証なら不要）。→ [security.md](./security.md)
- 認可忘れ：`Post.find` だと他人のも引ける。`current_user.posts.find` に。

## 関連
[routing.md](./routing.md) / [model.md](./model.md) / [view.md](./view.md) / [api_mode.md](./api_mode.md)
