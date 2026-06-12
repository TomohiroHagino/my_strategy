# ビュー（Rails 6）

## ひとことで言うと
`app/views` に置く `.html.erb` がレスポンスHTMLを組み立てる層。コントローラで作った `@変数` を使い、`<%= %>` で値を出力、`<% %>` でロジックを実行する。フォームは `form_with`、リンクは `link_to`/`button_to` で生成する。

## 役割・なぜ必要か
コントローラが用意したインスタンス変数を、ヘルパー（[helper.md](./helper.md)）で安全にHTML化して画面にする。共通の枠はレイアウト、繰り返し断片はpartialに切り出す（詳細は [partial_layout.md](./partial_layout.md)）。

## 基本の書き方（コード）
```erb
<%# app/views/posts/index.html.erb %>
<h1>記事一覧</h1>

<% @posts.each do |post| %>           <%# <% %> は出力しない（ロジックだけ） %>
  <article>
    <h2><%= link_to post.title, post_path(post) %></h2>   <%# <%= %> はHTMLに出力 %>
    <p><%= post.body %></p>            <%# 文字列は自動エスケープされXSS安全 %>
    <%= link_to "編集", edit_post_path(post) %>
    <%= link_to "削除", post_path(post),
          method: :delete,             <%# rails-ujs がDELETE送信に変換 %>
          data: { confirm: "本当に削除しますか？" } %>
  </article>
<% end %>
```
```erb
<%# app/views/posts/new.html.erb : フォーム %>
<%# Rails 6.0 の form_with は既定 remote:true（ajax送信）。同期送信したい場合は local: true 必須 %>
<%# Rails 6.1 で local が既定に変更された。6.0 でも明示しておくと安全 %>
<%= form_with model: @post, local: true do |f| %>
  <% if @post.errors.any? %>
    <ul>
      <% @post.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>           <%# 失敗時 render :new で再表示される %>
      <% end %>
    </ul>
  <% end %>

  <%= f.label :title %>
  <%= f.text_field :title %>
  <%= f.label :body %>
  <%= f.text_area :body %>
  <%= f.submit "保存" %>
<% end %>

<%# 単発のPOSTボタン（form_with不要なとき） %>
<%= button_to "公開", publish_post_path(@post), method: :post %>

<%# アセット：画像はimage_tag、パスはasset_path（[assets.md]） %>
<%= image_tag "logo.png", alt: "ロゴ" %>
```

## 実務での使い方・定番パターン
- 一覧→詳細→フォームの3画面が基本。フォームは `form_with model:` でnew/edit共用にする（送信先とメソッドを自動判定）。
- バリデーション失敗時はコントローラで `render :new`（[controller.md](./controller.md)）。`@post.errors` をビューで表示する。
- 繰り返す断片は `_post.html.erb` に切り出し `render @posts` で描画（[partial_layout.md](./partial_layout.md)）。
- 整形・条件分岐の重いロジックはビューに書かず、ヘルパーへ寄せる（[helper.md](./helper.md)）。

## ハマりどころ / アンチパターン
- `form_with` のremote既定（Rails 6.0）：`local: true` を付け忘れると非同期送信になり、サーバが普通にHTMLを返してもページが切り替わらず「送信できない」ように見える。6.0では必ず `local: true` を付ける。
- `html_safe` / `raw` 乱用：ユーザー入力に使うとXSSになる。自動エスケープに任せ、`sanitize` を使う（[security.md](./security.md)）。
- ビューでN+1誘発：`@posts.each { |p| p.comments.count }` のように関連を都度叩くと大量クエリになる。コントローラ側で `includes`/カウンタキャッシュを用意する（[active_record.md](./active_record.md)）。

## 関連
[controller.md](./controller.md) / [helper.md](./helper.md) / [partial_layout.md](./partial_layout.md) / [assets.md](./assets.md) / [javascript.md](./javascript.md) / [security.md](./security.md) / [routing.md](./routing.md)
