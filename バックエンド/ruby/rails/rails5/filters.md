# フィルタ（before_action など）（Rails 5）

## ひとことで言うと
コントローラのアクション実行の**前後に共通処理を差し込む仕組み**。`before_action` / `after_action` / `around_action` をクラスに宣言する。

## 役割・なぜ必要か
- 「ログインチェック」「対象レコードの取得」「権限確認」など、複数アクションで共通する前処理を、各アクションに書かず1か所に集約するためにある。
- DRY（重複排除）と、認証・認可の「漏れ」防止のための仕組み。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!                       # 全アクションの前に実行
  before_action :set_post, only:   %i[show edit update destroy]
  before_action :require_admin, except: %i[index show]

  def show; end   # set_post 済みなので @post が使える

  private

  def set_post
    @post = current_user.posts.find(params[:id])          # 取得＋認可を兼ねる
  end

  def require_admin
    redirect_to root_path, alert: "権限がありません" unless current_user.admin?
  end
end
```
- `before_action` が `redirect_to` / `render` すると、**そのアクションは実行されない**（処理を止められる）。

## 実務での使い方・定番パターン
- **認証**：`ApplicationController` に `before_action :authenticate_user!` を置き、公開アクションだけ `skip_before_action` で外す。→ [auth.md](./auth.md)
- **対象取得**：`set_post` で `@post` を用意。`only:` で対象アクションを絞る。
- **`skip_before_action`** で親の filter を子で無効化（公開ページなど）。
- **APIモード**でも同じ。トークン認証の検証を `before_action` で行うのが定番。→ [api_mode.md](./api_mode.md)
- 共通 filter を複数コントローラで使うなら **Concern** に切り出す。→ [concern.md](./concern.md)

## ハマりどころ / アンチパターン
- **filter 内の `return` の誤解**：`before_action` で止めたいときは `redirect_to`/`render` する（それで後続が止まる）。`return` だけでは止まらない場合がある。
- **filter のかけ忘れ・skip しすぎ**：`skip_before_action` で認証を外しすぎて公開すべきでないページが露出する。
- **filter に重い処理**：全アクションに効く `before_action` でDBを何度も叩くと全体が遅くなる。
- **順序依存**：`set_post` の後に `require_owner` を置く等、宣言順に実行されることを意識する。
- **Rails 5 の用語**：旧 `before_filter` は非推奨。**`before_action`** を使う（filter は概念名として残る）。

## 関連
[controller.md](./controller.md) / [auth.md](./auth.md) / [concern.md](./concern.md)
