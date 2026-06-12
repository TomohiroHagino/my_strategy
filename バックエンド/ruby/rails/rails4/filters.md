# フィルタ（before_filter → before_action）（Rails 4）

## ひとことで言うと
コントローラのアクションの**前後に共通処理を差し込む仕組み**。Rails 4 は **`before_filter` から `before_action` への過渡期**で、両方が動く（新規は `before_action` を使う）。

## 役割・なぜ必要か
- 認証チェック・対象レコードの取得・権限確認など、複数アクションで共通する処理を1箇所にまとめ、各アクションを薄く保つためにある。
- 「ログインしていなければ弾く」のような横断的関心事をアクション本体から分離する。

## 基本の書き方（コード）
```ruby
class PostsController < ApplicationController
  before_action :authenticate_user!                       # 全アクション前
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  after_action  :log_access, only: [:show]
  around_action :with_timing

  def show; end

  private

  def set_post
    @post = current_user.posts.find(params[:id])
  end

  def log_access
    Rails.logger.info("viewed post #{@post.id}")
  end

  def with_timing
    start = Time.current
    yield
    Rails.logger.info("took #{Time.current - start}s")
  end
end
```

### before_filter → before_action（過渡期の対応）
| 旧（〜Rails 3 / 4.0 でも動く） | 新（Rails 4.0〜推奨） |
|---|---|
| `before_filter` | `before_action` |
| `after_filter`  | `after_action`  |
| `around_filter` | `around_action` |
| `skip_before_filter` | `skip_before_action` |
| `prepend_before_filter` | `prepend_before_action` |

- `*_filter` は **Rails 4 で非推奨（deprecated）扱い**になり、**Rails 5 で削除**された。Rails 4 のうちに `*_action` へ寄せる。
- 既存コードに `before_filter` が残っていても 4 では動く。混在させず統一する。

## 実務での使い方・定番パターン
- **`only:` / `except:`** で対象アクションを限定。
- **`set_xxx`** で対象レコード取得を共通化（scaffold もこの形を生成）。
- **認証**は `before_action :authenticate_user!`（devise）や自前の `require_login`。→ [auth.md](./auth.md)
- **中断**：フィルタ内で `redirect_to` / `render` を呼ぶと、そのアクションは実行されずに止まる（認証失敗時の弾き）。
  ```ruby
  def require_login
    redirect_to login_path unless current_user   # ここで応答すればアクションは走らない
  end
  ```
- 共通フィルタは `ApplicationController` に置いて継承。

## ハマりどころ / アンチパターン
- **`before_filter` を新規で書く**：4 では非推奨。`before_action` に。
- **フィルタで render/redirect 後に処理が続くと二重応答**：`DoubleRenderError`。フィルタで応答したら本体は走らない前提で書く。
- **`skip_before_action` の対象漏れ**：継承先で特定アクションだけ外したい時にメソッド名を間違える。
- **フィルタに重いロジックを詰める**：副作用の多い処理はService/Jobへ。フィルタは「準備とガード」に留める。
- **順序依存**：`before_action` は宣言順に実行される。`set_post` の後に `authorize` を置くなど順序に注意。

## 関連
[controller.md](./controller.md) / [auth.md](./auth.md) / [concern.md](./concern.md)
