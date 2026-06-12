# partial と layout（Rails 7）

## ひとことで言うと
**partial** は `_` 始まりのファイル名で定義する再利用可能なビュー部品、**layout** はページ全体の外枠（ヘッダ/フッタ等）で、本文を `yield` で差し込む。どちらもビューの重複を減らす仕組み。

## 役割・なぜ必要か
- **partial**：フォーム・一覧の行・カードなど繰り返す断片を切り出して使い回す。1画面の中でも、複数画面間でも共有できる。
- **layout**：全ページ共通の `<html>`〜`<body>`、ナビ、フラッシュ表示などを1か所に置き、各ビューの中身だけを差し替える。既定は `app/views/layouts/application.html.erb`。
- 役割分担で「中身（partial）」と「外枠（layout）」を分け、DRYに保つ。

## 基本の書き方（コード）
```erb
<%# app/views/posts/_post.erb（partial：ファイル名は _ 始まり） %>
<article>
  <h2><%= post.title %></h2>
  <p><%= post.body %></p>
</article>
```

```erb
<%# 呼び出し側 %>
<%= render "post", post: @post %>            <%# = render partial: "posts/post", locals: { post: @post } %>
<%= render partial: "shared/flash", locals: { message: notice } %>

<%# コレクション描画：_post を各要素に対し繰り返す。post というローカル変数が自動で入る %>
<%= render @posts %>                          <%# モデル名から _post を推測（最も簡潔） %>
<%= render partial: "post", collection: @posts %>
<%= render partial: "post", collection: @posts, as: :post %>
```

```erb
<%# app/views/layouts/application.html.erb（layout） %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for?(:title) ? yield(:title) : "MyApp" %></title>
    <%= csrf_meta_tags %>
    <%= yield :head %>                        <%# 名前付きで差し込み %>
  </head>
  <body>
    <%= render "shared/nav" %>
    <main><%= yield %></main>                 <%# 各アクションのビュー本文がここに入る %>
  </body>
</html>
```

```erb
<%# 各ビュー側で content_for に詰めて layout の yield(:xxx) へ渡す %>
<% content_for :title, "記事一覧" %>
<% content_for :head do %><meta name="robots" content="noindex"><% end %>
```

## 実務での使い方・定番パターン
- **`render "form"`** で `new`/`edit` のフォームを共有（`_form.html.erb`）。Rails の定番。
- **コレクション描画**：`render @posts` は `_post` partial を要素数ぶん描く。`<%= render @posts %>` が最短形。要素間に区切りを入れるなら `spacer_template:`。
- **`content_for` + `yield(:name)`** で、ページごとにタイトルや `<head>` 追記を layout へ流し込む。
- **layout の切り替え**：コントローラで `layout "admin"`、アクション単位なら `render layout: "modal"`。
- **partial にもローカル変数を明示で渡す**：`locals:` で渡した変数だけが partial 内で使える（partial はインスタンス変数 `@post` に依存させず、ローカル変数で受けると再利用性が上がる）。

## ハマりどころ / アンチパターン
- **コレクション描画の N+1**：`render @posts` の各 partial が `post.user.name` などを参照すると要素数ぶんクエリが飛ぶ。コントローラ側で `includes` して eager load する。→ [active_record.md](./active_record.md)
- **ローカル変数の渡し忘れ**：`render "post"`（locals なし）で partial 内が `post` を参照すると `undefined local variable`。`locals:` か `render @post` で必ず渡す。
- **partial で `@インスタンス変数` に依存**：呼び出し元が変わると壊れる。partial は `locals` で受ける設計にする。
- **`render @posts` で空配列**：`@posts` が空だと**何も描画されず nil 返却**。「該当なし」表示は `if @posts.any?` で別途。
- **`content_for` の重複呼び出し**：同名キーに複数回 `content_for` すると**追記され連結**される（上書きではない）。意図しない重複表示に注意。

## 関連
[view.md](./view.md) / [active_record.md](./active_record.md)
