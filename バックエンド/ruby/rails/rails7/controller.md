# コントローラ（Controller）（Rails 7）

## ひとことで言うと
リクエストを受け取り、**モデルを呼んでデータを用意し、ビュー（HTML）か JSON を返す「司令塔」**。`ApplicationController` を継承し、`app/controllers/` に置く。

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
      render :new, status: :unprocessable_entity   # 7系のフォーム再描画の定石
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

## 実務での使い方・定番パターン
- **before_action** で共通処理（認証・対象レコード取得）を切り出す。→ [filters.md](./filters.md)
- **Strong Parameters**（`post_params`）で許可キーだけ通す。→ [strong_parameters.md](./strong_parameters.md)
- **`current_user.posts.find(...)`** のように常にログインユーザのスコープで引くと、他人のリソース操作を自然に防げる（認可の基本）。
- **render と status**：作成/更新失敗時は `status: :unprocessable_entity` を付ける（Turboがフォームエラーを正しく再描画するため）。
- **`rescue_from`** で例外を共通ハンドリング（`RecordNotFound` → 404 等）。
- API なら `ActionController::API` 継承＋ `render json:`。→ 別途 api 項目（予定）

## ハマりどころ / アンチパターン
- **二重 render/redirect**（`DoubleRenderError`）：1アクションで応答は1回。分岐後に `return` を忘れない。
- **Fat Controller**：分岐・計算が増えたらモデル/Serviceへ。
- **Strong Parameters 漏れ**：permit し忘れたキーは黙って無視される＝「保存したのに入らない」。
- **redirect 後に status を付け忘れ**てTurboのフォームエラーが出ない（7系の頻出）。
- 認可忘れ：`Post.find` だと他人のも引ける。`current_user.posts.find` に。

## 関連
[routing.md](./routing.md) / [model.md](./model.md) / [view.md](./view.md)
