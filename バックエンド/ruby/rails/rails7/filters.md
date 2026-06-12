# フィルタ（before_action 等）（Rails 7）

## ひとことで言うと
コントローラのアクション実行の**前後・周囲に共通処理を差し込む仕組み**。`before_action` / `after_action` / `around_action` の3種。認証チェックや対象レコード取得をアクションから切り出すために使う。

## 役割・なぜ必要か
- 複数アクションで繰り返す共通処理（ログイン確認、`@post` の取得など）を1か所に集約し、各アクションを薄く保つ（DRY）。
- **before_action** が最頻出。アクションの手前で割り込み、認証で弾く・対象を用意する、といった「前処理」を担う。
- フィルタ内で `redirect_to` / `render` を呼ぶと**そこで処理が止まり（halt）、本来のアクションは実行されない**。これが認証ガードの肝。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  # 全アクション前にログイン確認
  before_action :authenticate_user!
  # 一部アクションだけ対象レコードを取得
  before_action :set_post, only: %i[show edit update destroy]
  # 後処理・周囲処理
  after_action  :log_access, only: :show
  around_action :with_time_zone

  def show; end          # set_post 済みなので @post が使える
  def index
    @posts = current_user.posts.recent
  end

  private

  def set_post
    @post = current_user.posts.find(params[:id])  # 取得＋認可を兼ねる
  end

  def authenticate_user!
    redirect_to login_path, alert: "ログインが必要です" unless current_user
    # ↑ ここで redirect すると以降のアクションは実行されない（halt）
  end

  def log_access
    Rails.logger.info("showed post=#{@post.id}")
  end

  def with_time_zone
    Time.use_zone(current_user&.time_zone || "Tokyo") { yield }  # yield の前後で囲む
  end
end
```

```ruby
# 親で掛けたフィルタを子で外す
class PublicPostsController < PostsController
  skip_before_action :authenticate_user!, only: %i[index show]
end
```

## 実務での使い方・定番パターン
- **`set_xxx` で対象取得**：`set_post` のように `@post` を用意し、`only:` で必要なアクションだけに掛けるのが定石。`current_user.posts.find` にすると認可も自然に効く（[controller.md](./controller.md)）。
- **`authenticate_user!` で認証ガード**：未ログインなら `redirect_to` して halt。`ApplicationController` に置いて全体へ、例外的に外したい所だけ `skip_before_action`。
- **`only:` / `except:` で対象を絞る**：付け忘れると全アクションに掛かる。基本は `only:` で明示するのが安全。
- **実行順序**：`before_action` は**宣言した順**に上から実行。`set_post` より先に `authenticate_user!` を宣言する（先に認証で弾く）。`after_action` は逆順、`around_action` は `yield` で前後を囲む。
- **継承での共有**：`ApplicationController` に共通フィルタを置けば子コントローラ全体へ適用される。

## ハマりどころ / アンチパターン
- **順序ミス**：`set_post` を `authenticate_user!` より前に宣言すると、未ログインでもレコード取得が走る。認証系は先頭に。
- **`only:` 付け忘れ**：`set_post` を全アクションに掛けると `index`/`new` で `params[:id]` が無く `RecordNotFound` になる。
- **halt の理解不足**：before_action 内で `redirect`/`render` を呼べばアクションは実行されない。一方、条件分岐で呼ばなかった場合は素通りするので「ガードしたつもりが通っていた」事故に注意。`return` の有無ではなく**応答を出したかどうか**で halt する。
- **二重 render**：フィルタとアクションの両方で render/redirect すると `DoubleRenderError`。フィルタで応答したらアクションは実行されない点を前提に組む。
- **`skip_before_action` のキー不一致**：親と違うメソッド名やオプションだと `ArgumentError`。Rails 7 は既定で厳密（`raise: true` 相当）。

## 関連
[controller.md](./controller.md)
