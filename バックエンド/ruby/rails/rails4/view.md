# ビュー（View）（Rails 4）

## ひとことで言うと
コントローラが用意したデータ（インスタンス変数）を受け取り、**HTMLを組み立てて返すテンプレート**。`app/views/` 配下に `アクション名.html.erb` として置く。

## 役割・なぜ必要か
- MVCの「見た目」担当。コントローラが詰めた `@posts` 等を埋め込んで最終的なHTMLを作る。
- ロジック（条件分岐・整形）をビューに書きすぎると読めなくなるので、整形は **ヘルパー**、判断は **モデル/コントローラ** に寄せ、ビューは表示に専念する。

## 基本の書き方（コード）
```erb
<%# app/views/posts/index.html.erb %>
<h1>記事一覧</h1>

<% @posts.each do |post| %>
  <article>
    <h2><%= link_to post.title, post %></h2>
    <p><%= truncate(post.body, length: 100) %></p>
    <%= link_to "編集", edit_post_path(post) %>
    <%= link_to "削除", post, method: :delete,
          data: { confirm: "削除しますか？" } %>   <%# jquery_ujs が data-method/data-confirm を処理 %>
  </article>
<% end %>

<%= link_to "新規作成", new_post_path %>
```
- `<%= %>` は出力あり、`<% %>` は出力なし（制御だけ）。
- `<%= %>` は **自動でHTMLエスケープ**される（XSS対策）。生HTMLを出したい時だけ `raw` / `html_safe`（要注意）。→ [security.md](./security.md)

## フォーム
```erb
<%= form_for @post do |f| %>
  <% if @post.errors.any? %>
    <ul>
      <% @post.errors.full_messages.each do |msg| %>
        <li><%= msg %></li>
      <% end %>
    </ul>
  <% end %>

  <%= f.label :title %>
  <%= f.text_field :title %>
  <%= f.text_area :body %>
  <%= f.submit %>
<% end %>
```
- Rails 4 は **`form_for` / `form_tag`** が主役（`form_with` は5.1から。4には無い）。
- `form_for @post` は `@post` の new/persisted を見て POST/PATCH を自動で振り分け、Strong Parameters の `params[:post]` に入る形を作る。

## 実務での使い方・定番パターン
- **partial** で共通部分を切り出す（`render "form"` / コレクション描画 `render @posts`）。→ [partial_layout.md](./partial_layout.md)
- **layout**（`app/views/layouts/application.html.erb`）で共通の枠（ヘッダ/フッタ/CSS・JS読み込み）を定義。→ [partial_layout.md](./partial_layout.md)
- **Ajax 部分更新**は `*.js.erb`（SJR）でサーバがJSを返して書き換える。Rails 4 の定番。→ [javascript.md](./javascript.md)
- **整形はヘルパーへ**：`<%= post_status_label(post) %>` のように。ビューに `if` を増やさない。→ [helper.md](./helper.md)
- テンプレートエンジンは ERB の他に **Haml / Slim** も現場で使われる。
- `content_for` / `yield(:head)` でレイアウトの差し込み口を作る。

## ハマりどころ / アンチパターン
- **`html_safe` / `raw` の乱用**：ユーザ入力に付けるとXSS。不安なら `sanitize`。→ [security.md](./security.md)
- **ビューにロジックを書きすぎ**：複雑な `if` ネストはヘルパー/プレゼンターへ。
- **N+1**：ビューのループ内で `post.user.name` を呼ぶとN+1。コントローラで `includes` する。→ [active_record.md](./active_record.md)
- **`form_with` を探す**：4には無い（`form_for`/`form_tag` を使う）。
- **Turbolinks Classic で JS が動かない**：`<head>` 内の `<script>` が再評価されない。`page:load` で再初期化。→ [javascript.md](./javascript.md)

## 関連
[controller.md](./controller.md) / [helper.md](./helper.md) / [partial_layout.md](./partial_layout.md) / [javascript.md](./javascript.md)
