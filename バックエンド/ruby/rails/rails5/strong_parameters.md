# Strong Parameters（Rails 5）

## ひとことで言うと
フォームやAPIから送られた `params` のうち、**「受け取ってよいキーだけ」を明示的に許可してからモデルに渡す仕組み**。意図しない属性の一括代入を防ぐ。

## 役割・なぜ必要か
- `Post.new(params[:post])` のように生 params をそのまま渡すと、攻撃者が `admin: true` 等の余計なキーを混ぜて権限昇格できる（マスアサインメント脆弱性）。
- 「どのキーを通すか」をコントローラで宣言させ、許可外を弾くためにある。Rails 4 以降の標準防御。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    # ...
  end

  private

  def post_params
    params.require(:post)                       # :post キーが無ければ例外
          .permit(:title, :body, :published)    # 許可するキーだけ
  end
end
```
```ruby
# ネスト・配列・複合の例
params.require(:post).permit(
  :title,
  tag_ids: [],                                  # 配列
  comments_attributes: [:id, :body, :_destroy]  # accepts_nested_attributes_for 用
)
```

## 実務での使い方・定番パターン
- **`require` + `permit`** が基本形。`require` は「親キー必須」、`permit` は「子キー許可」。
- **配列は `key: []`**、**ネスト属性は `key_attributes: [...]`** で許可。
- **動的に許可キーを変える**（管理者だけ `role` を許可など）：
  ```ruby
  def user_params
    permitted = %i[name email]
    permitted << :role if current_user.admin?
    params.require(:user).permit(*permitted)
  end
  ```
- **APIモード**でも同じ。JSONボディも `params` に入るので permit する。→ [api_mode.md](./api_mode.md)

## ハマりどころ / アンチパターン
- **permit 漏れ**：許可し忘れたキーは**黙って無視**される＝「フォームで入力したのに保存されない」。これが最頻のハマり。
- **`permit!`（全許可）の使用**：マスアサインメント防御を自分で無効化することになる。基本使わない。
- **`require` の対象ミス**：`params.require(:post)` なのにフォームの name が `post[...]` でないと `ParameterMissing` 例外。
- **ネスト/配列の許可形を間違える**：`tag_ids` を `permit(:tag_ids)` にすると配列が通らない。`tag_ids: []` が正しい。
- **強い型付けではない**：permit は「キーの許可」だけで値の検証はしない。値の妥当性は**バリデーション**で行う。→ [model.md](./model.md)

## 関連
[controller.md](./controller.md) / [model.md](./model.md) / [security.md](./security.md)
