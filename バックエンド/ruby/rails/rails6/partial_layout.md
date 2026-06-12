# partial / layout（Rails 6）

## ひとことで言うと
partial はビューの部品（`_form.html.erb` 等）を切り出して再利用する仕組み。layout は全画面共通の外枠（ヘッダ・フッタ）で、各ビューを `yield` に差し込む。

## 役割・なぜ必要か
new と edit でほぼ同じフォームを2回書くのは重複。共通部分を `_form.html.erb` に切り出して両方から `render` すれば一箇所の修正で済む。layout は `<html>` から `<body>` までの共通枠を一度だけ書く場所。

## 基本の書き方（コード）
```erb
<%# app/views/posts/new.html.erb %>
<%= render "form", post: @post %>

<%# app/views/posts/_form.html.erb （先頭アンダースコアが partial の印） %>
<%= form_with model: post, local: true do |f| %>
  <%= f.text_field :title %>
<% end %>
```
明示指定・locals・コレクション。
```erb
<%= render partial: "post", locals: { post: @post } %>
<%= render @posts %>   <%# 各要素を _post.html.erb でレンダリング %>
<%= render partial: "post", collection: @posts %>
```
layout（共通枠）と `yield` / `content_for`。
```erb
<%# app/views/layouts/application.html.erb %>
<head><title><%= content_for?(:title) ? yield(:title) : "MyApp" %></title></head>
<body><%= yield %></body>   <%# 各アクションのビューがここに入る %>

<%# 個別ビュー側 %>
<% content_for :title, "投稿一覧" %>
```
layout を使わない・差し替える場合。
```ruby
render :show, layout: false
render :show, layout: "admin"
```

## 実務での使い方・定番パターン
- `_form.html.erb` を new / edit で共有する（最頻出）。
- 一覧は `render @posts` のコレクションレンダリングで簡潔に書く。
- 共通の通知バーやサイドバーは partial に切り出し、layout から `render "shared/nav"` で読む。

## ハマりどころ / アンチパターン
- `render @posts` で各 partial が関連を呼ぶと N+1。コントローラ側で `includes` する（[active_record.md](./active_record.md)）。
- `render "post"` で partial 内が必要とする変数を locals で渡し忘れ、`nil` や `NameError` になる。`locals:` で明示する。
- layout に `<%= yield %>` を書き忘れると本文が出ない。
- partial 名のアンダースコアは render では付けない（`render "form"` で `_form.html.erb`）。

## 関連
[view.md](./view.md) / [helper.md](./helper.md) / [controller.md](./controller.md) / [active_record.md](./active_record.md) / [assets.md](./assets.md)
