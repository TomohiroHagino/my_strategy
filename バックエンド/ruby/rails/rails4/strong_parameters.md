# Strong Parameters（Rails 4）

## ひとことで言うと
フォーム等から送られた `params` のうち、**保存に使ってよいキーをコントローラ側で明示的に許可する仕組み**。Rails 4 で**標準**になり、旧来の `attr_accessible`（モデル側で許可）は本体から外れた。

## 役割・なぜ必要か
- 悪意ある送信者が `params` に余分なキー（例：`admin: true`）を紛れ込ませて更新する **mass assignment 攻撃**を防ぐためにある。
- Rails 3 まではモデルに `attr_accessible :title` と書いて許可していたが、Rails 4 では**コントローラ（リクエストの入口）で許可する**設計に変わった。許可場所がモデルからコントローラに移った点が要。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)   # 直接 params[:post] を渡すと例外になる
    @post.save
  end

  private

  def post_params
    params.require(:post).permit(:title, :body, :published)
  end
end
```
- `require(:post)` … `params[:post]` が無ければ `ParameterMissing` 例外（必須）。
- `permit(:title, ...)` … 許可するキーを列挙。許可外のキーは**黙って無視**される。

### 配列・ネスト
```ruby
# 配列（チェックボックス group など）
params.require(:post).permit(:title, tag_ids: [])

# ネストした属性（accepts_nested_attributes_for）
params.require(:post).permit(:title, comments_attributes: [:id, :body, :_destroy])

# 任意キー（ハッシュ丸ごと許可。値が動的なとき）
params.require(:setting).permit(preferences: {})
```

## 実務での使い方・定番パターン
- **`xxx_params` private メソッドに切り出す**のが定番（scaffold もこの形を生成）。
- **作成と更新で許可キーを変える**：更新ではパスワードを任意にする等、メソッドを分ける/条件分岐する。
- **`attr_accessible` は書かない**：Rails 4 では `protected_attributes` gem を入れない限り使えない。レガシー移行時のみ gem で延命する。
- ネスト保存は `accepts_nested_attributes_for` ＋ `*_attributes: [...]` の許可をセットで。
- API でも同じ。`params.require(:user).permit(...)` で守る。

## ハマりどころ / アンチパターン
- **permit 漏れ**：許可し忘れたキーは保存されない＝「フォームに入れたのにDBに入らない」。最頻のハマり。
- **`require` 対象が無い**：`params[:post]` が来ない形（トップレベルにキーがある）だと `ParameterMissing`。フォームの `name` 属性とモデル名を合わせる。
- **配列・ネストの書き方ミス**：`permit(:tag_ids)`（スカラ）では配列が通らない。`permit(tag_ids: [])` と書く。
- **`permit!`（全許可）**：危険。`params.permit!` は mass assignment 防御を無効化する。使わない。
- **古い記事の `attr_accessible`**：Rails 3 の解説をそのまま真似るとモデルに書いて動かない。許可はコントローラで。

## 関連
[controller.md](./controller.md) / [model.md](./model.md) / [security.md](./security.md)
