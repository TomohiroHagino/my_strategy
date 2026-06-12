# partial / layout（Rails 4）

## ひとことで言うと
**partial** はビューの一部を `_form.html.erb` のように切り出して再利用する部品、**layout** はヘッダ/フッタなど全ページ共通の外枠テンプレート。

## 役割・なぜ必要か
- 同じHTML断片（フォーム・カード・一覧の行）を複数ビューで使い回すために partial がある。重複を減らし変更を1箇所にまとめる。
- 全ページで共通の `<html>`〜`<body>`、CSS/JS読み込み、ナビゲーションを1つに集約するために layout がある。

## 基本の書き方（コード）

### partial
```erb
<%# app/views/posts/_post.html.erb（先頭アンダースコアが目印） %>
<article>
  <h2><%= post.title %></h2>
  <p><%= post.body %></p>
</article>
```
```erb
<%# 呼び出し側 %>
<%= render "post", post: @post %>           <%# 単体 %>
<%= render partial: "post", locals: { post: @post } %>  <%# 明示版 %>
<%= render @posts %>                         <%# コレクション：_post を各要素で描画 %>
<%= render @post %>                          <%# 1件：_post を post ローカルで描画 %>
```

### layout
```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <title>MyApp</title>
  <%= stylesheet_link_tag "application", media: "all" %>
  <%= javascript_include_tag "application" %>   <%# Sprockets が結合配信 %>
  <%= csrf_meta_tags %>                         <%# CSRFトークン %>
</head>
<body>
  <% flash.each do |type, msg| %>
    <div class="flash flash-<%= type %>"><%= msg %></div>
  <% end %>

  <%= yield %>                                  <%# 各アクションのビューがここに入る %>
</body>
</html>
```
- `javascript_include_tag "application"` は Sprockets の `app/assets/javascripts/application.js` を読む。→ [assets.md](./assets.md)

## 実務での使い方・定番パターン
- **フォームの共通化**：`new` と `edit` で `render "form"`。
- **コレクション描画**：`render @posts` は `_post` を各要素で繰り返す（`render partial: "post", collection: @posts` の短縮）。`post_counter` ローカルも自動で使える。
- **layout の切り替え**：コントローラで `layout "admin"` や `render layout: "modal"`。
- **`content_for`**：partial/ビューから layout の差し込み口へ流す。
  ```erb
  <% content_for :head do %><meta ...><% end %>
  <%# layout 側 %><%= yield :head %>
  ```
- **共通パーシャルは `app/views/shared/`** に置く慣習（`render "shared/header"`）。

## ハマりどころ / アンチパターン
- **partial 名のアンダースコア**：ファイルは `_post.html.erb`、呼ぶ時は `render "post"`（アンダースコア無し）。
- **locals 渡し忘れ**：partial 内で使う変数は `locals:` で明示的に渡す（インスタンス変数依存にしない方が再利用しやすい）。
- **コレクション描画で N+1**：`render @posts` の中で関連を引くとN+1。コントローラで `includes`。→ [active_record.md](./active_record.md)
- **layout のJS/CSS読み込み忘れ**：`javascript_include_tag` / `stylesheet_link_tag` が無いとアセットが効かない。
- **Turbolinks Classic**：layout の `<head>` 内 `<script>` は遷移時に再評価されない。→ [javascript.md](./javascript.md)

## 関連
[view.md](./view.md) / [helper.md](./helper.md) / [assets.md](./assets.md) / [session_cookie_flash.md](./session_cookie_flash.md)
