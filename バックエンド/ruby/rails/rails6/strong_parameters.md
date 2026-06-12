# Strong Parameters（Rails 6）

## ひとことで言うと
フォームから来た `params` のうち、保存を許可するキーだけを明示する仕組み。許可していないキーは無視され、勝手なカラム更新（mass assignment）を防ぐ。

## 役割・なぜ必要か
`Post.new(params[:post])` をそのまま許すと、フォームに無い `admin: true` 等を送られて更新される危険がある。`permit` で許可キーを列挙し、`require` で必須の親キーを指定することで、想定外の属性を弾く。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    if @post.save
      redirect_to @post, notice: "作成しました"
    else
      render :new
    end
  end

  private

  def post_params
    params.require(:post).permit(:title, :body, :published)
  end
end
```
配列・ネストした属性の許可。
```ruby
# 値が配列のキー（チェックボックス群など）は [] を付ける
params.require(:post).permit(:title, tag_ids: [])

# ネストしたハッシュはキーを列挙
params.require(:user).permit(:name, address: [:zip, :city])

# accepts_nested_attributes_for と組み合わせる場合
params.require(:post).permit(:title, comments_attributes: [:id, :body, :_destroy])
```

## 実務での使い方・定番パターン
- アクションごとではなく `xxx_params` という private メソッドにまとめ、create / update から共通で呼ぶ。
- 許可キーはフォームの入力項目と一致させて管理する。
- `permit!`（全許可）は使わない。便利だが mass assignment 脆弱性そのものになる。

## ハマりどころ / アンチパターン
- `permit` に書き忘れたキーは **エラーにならず黙って無視**される。「保存したのに値が入らない」原因の大半はこれ。
- 配列・ネストは `tags: []` や `address: [:zip]` の形にしないと弾かれる。スカラーと同じ書き方では通らない。
- `params.require(:post)` で `:post` キーが無いと `ActionController::ParameterMissing` で 400。`params.expect` は Rails 8 の機能なので Rails 6 では使わない。
- 信頼できない属性をそのまま `permit` に並べると mass assignment の穴になる。`role` 等は別経路で更新する。

## 関連
[controller.md](./controller.md) / [model.md](./model.md) / [filters.md](./filters.md) / [security.md](./security.md) / [active_record.md](./active_record.md)
