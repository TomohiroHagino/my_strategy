# ヘルパー（Rails 6）

## ひとことで言うと
`app/helpers` 配下に置く、ビューから呼ぶためのメソッド集。表示の整形ロジックをビューやモデルから切り出す場所。

## 役割・なぜ必要か
ERB に長い条件分岐や整形処理を書くとビューが読めなくなる。整形処理をメソッド化して `app/helpers` に置くと、ビューは `<%= post_status_label(post) %>` のように呼ぶだけで済む。Rails 6 ではヘルパーは全ビューでグローバルに使える（`include_all_helpers` 既定 true）。

## 基本の書き方（コード）
```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def post_status_label(post)
    post.published? ? "公開" : "下書き"
  end
end
```
```erb
<%# app/views/posts/show.html.erb %>
<%= post_status_label(@post) %>
<%= link_to "編集", edit_post_path(@post) %>
<%= number_to_currency(@post.price, unit: "￥") %>
<%= time_ago_in_words(@post.created_at) %>前
```
コントローラのメソッドをビューでも使いたいときは `helper_method`。
```ruby
class ApplicationController < ActionController::Base
  helper_method :current_user
  private
  def current_user
    @current_user ||= User.find_by(id: session[:user_id])
  end
end
```
HTML を組み立てるときは `tag` / `content_tag`、文字列をそのまま HTML として出すときは `html_safe`（または `sanitize`）。
```ruby
def badge(text)
  content_tag(:span, text, class: "badge")  # 自動エスケープされ安全
end
```

## 実務での使い方・定番パターン
- 表示整形（日付・金額・ラベル・CSSクラス分岐）はヘルパーに置く。
- 共通の `ApplicationHelper` には全画面で使う汎用整形を、画面固有は `PostsHelper` 等に置く。
- ビジネスロジックはモデルや decorator（draper等）に置き、ヘルパーは「表示の都合」に限定する。

## ハマりどころ / アンチパターン
- ヘルパーは全ビューでグローバル。`PostsHelper` のメソッドも `users/` のビューから呼べてしまい、同名メソッドが衝突する。汎用名（`title` 等）は避ける。
- ロジックを詰め込みすぎてヘルパーが肥大化する。条件分岐が増えたら decorator やモデルメソッドへ移す。
- ユーザー入力を `html_safe` でそのまま出すと XSS。エスケープ済みの安全な文字列にのみ使う（[security.md](./security.md)）。

## 関連
[view.md](./view.md) / [partial_layout.md](./partial_layout.md) / [controller.md](./controller.md) / [security.md](./security.md) / [service_form.md](./service_form.md)
