# Strong Parameters（Rails 7）

## ひとことで言うと
コントローラで受け取った `params` のうち、**モデルへ一括代入（mass assignment）してよいキーだけを明示的に許可する仕組み**。`params.require(:x).permit(:a, :b)` の形で使う。

## 役割・なぜ必要か
- `User.new(params[:user])` のように生の params をそのまま渡すと、フォームに無い `admin` などのキーまで送りつけられて更新される（**mass assignment 脆弱性**）。これを防ぐのが目的。
- 「**このアクションでは、このキーだけ書き換えてよい**」という許可リスト（allowlist）をコントローラ側で宣言する。許可していないキーは黙って捨てられる。
- Rails 4 以降、`attr_accessible`（モデル側で制御）からコントローラ側の Strong Parameters へ移行した。許可の責務は入口（コントローラ）にある。

## 基本の書き方（コード）
```ruby
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user, notice: "作成しました"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    # require: そのキーが無ければ ParameterMissing 例外
    # permit : 許可したキーだけ通す（admin は送られても無視される）
    params.require(:user).permit(:name, :email)
  end
end
```

```ruby
# 配列・ネストの許可
def post_params
  params.require(:post).permit(
    :title, :body,
    tag_ids: [],                          # 配列パラメータは [] を付ける
    category_ids: [],
    images: [],                           # スカラー値の配列
    options: {},                          # キー不定のハッシュ全許可（中身は信用できる前提のみ）
    comments_attributes: [:id, :body, :_destroy]  # nested attributes
  )
end
```

## 実務での使い方・定番パターン
- **`require` の使い分け**：トップレベルキーが必須なら `require`。任意なら `params.permit(:q, :page)` のように `require` なしで `permit` だけ。
- **`tag_ids: []`** は `has_many through` のチェックボックス選択で頻出。配列は必ず `: []` を付ける（付け忘れると無視される）。
- **`accepts_nested_attributes_for`** とセットで `xxx_attributes: [...]` を許可。子レコードを同一フォームで作成/更新/削除する場合、`:id` と `:_destroy` も許可キーに含める。
  ```ruby
  # app/models/post.rb
  class Post < ApplicationRecord
    has_many :comments
    accepts_nested_attributes_for :comments, allow_destroy: true
  end
  ```
- **共通化**：`new`/`create`/`update` で同じ `xxx_params` を private メソッドに切り出して共有する（[controller.md](./controller.md) の定石）。
- 値による分岐が必要なら `permit` 後に手で詰める（例：管理者のときだけ `:role` を許可）。
  ```ruby
  def user_params
    permitted = params.require(:user).permit(:name, :email)
    permitted = permitted.merge(role: params[:user][:role]) if current_user.admin?
    permitted
  end
  ```

## ハマりどころ / アンチパターン
- **permit 漏れ**：許可し忘れたキーは例外にならず**黙って捨てられる**。「フォームに入力したのに保存されない」の典型原因。まずここを疑う。
- **配列の `[]` 付け忘れ**：`tag_ids` を `permit(:tag_ids)`（スカラー扱い）にすると配列が通らず空になる。配列は `tag_ids: []`。
- **`permit!`（全許可）は原則禁止**：`params.require(:user).permit!` は全キーを通すので mass assignment 脆弱性そのもの。外部入力には絶対に使わない。
- **ネスト構造の取り違え**：`permit(user: [:name])` と `require(:user).permit(:name)` は通る形が違う。送られてくる params の構造（フォーム/JSON）に合わせる。
- **strong params はバリデーションではない**：許可しただけで値の正当性は保証されない。中身の検証は**モデルの validates** で行う。→ [model.md](./model.md)

## 関連
[controller.md](./controller.md) / [model.md](./model.md)
