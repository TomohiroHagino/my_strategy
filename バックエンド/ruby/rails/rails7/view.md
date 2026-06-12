# ビュー（View）（Rails 7）

## ひとことで言うと
ユーザーに見せる **HTMLを組み立てる層**。`app/views/` に置き、標準テンプレートは **ERB**（`<%= %>` で値を出力、`<% %>` でロジック）。

## 役割・なぜ必要か
- MVCの「見た目」担当。コントローラが用意したインスタンス変数（`@post` 等）を受け取って表示する。
- アクション名と同名のテンプレートが自動で描画される（`PostsController#show` → `app/views/posts/show.html.erb`）。規約で結びつくので明示指定が要らない。

## 基本の書き方（コード）
```erb
<%# app/views/posts/show.html.erb %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>

<%# フォーム：CSRFトークンも自動で入る %>
<%= form_with model: @post do |f| %>
  <%= f.label :title %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>

<%# 部品の再利用 %>
<%= render "shared/flash" %>
<%= render partial: "post", collection: @posts %>
```

## 実務での使い方・定番パターン
- **partial（部分テンプレート）** で共通部品を切り出す（`_form.html.erb` を new/edit で共有）。→ [partial_layout.md](./partial_layout.md)
- **layout** で全ページ共通の枠（ヘッダ/フッタ）を持ち、各ページは `yield` に入る。
- **form_with** がフォームの標準（7系は既定で通常送信＝localが既定。Turbo連携も自然）。
- 表示用のちょっとしたロジックは**ヘルパー**へ（ビューにRubyを書きすぎない）。→ [helper.md](./helper.md)
- Hotwire を使うと、ビューの一部を `turbo_frame` / `turbo_stream` で部分更新できる。→ [hotwire.md](./hotwire.md)

## ハマりどころ / アンチパターン
- **ビューでDBアクセス**（`@posts.each { |p| p.comments.count }`）→ N+1の温床。コントローラ側で `includes` する。→ [active_record.md](./active_record.md)
- **`raw` / `html_safe` の乱用**：ERBは既定でHTMLエスケープして**XSSを防ぐ**。ユーザー入力に `raw` を付けると穴になる。→ [security.md](./security.md)
- **ロジックの詰め込み**：分岐だらけのビューは読めない。helper / partial / presenter に逃がす。
- partial への**ローカル変数渡し忘れ**（`render "post", post: p`）。

## 関連
[controller.md](./controller.md) / [helper.md](./helper.md) / [hotwire.md](./hotwire.md)
