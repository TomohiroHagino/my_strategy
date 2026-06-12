# partial / layout（Rails 5）

## ひとことで言うと
**partial（部分テンプレート）** はビューの再利用できる断片、**layout** はページ全体の共通の枠（ヘッダ・フッタ・`yield`）。どちらも重複を減らすための仕組み。

## 役割・なぜ必要か
- 同じUI（記事カード・フォーム・ナビ）を複数ページで使い回すために partial がある。
- 全ページ共通の外枠（HTML骨格・メタタグ・読み込むJS/CSS）を1か所にまとめるために layout がある。

## partial の基本
```erb
<%# app/views/posts/_post.html.erb（ファイル名は _ 始まり） %>
<article>
  <h2><%= post.title %></h2>
  <p><%= post.body %></p>
</article>
```
```erb
<%# 呼び出し %>
<%= render "post", post: @post %>          <%# 単体 %>
<%= render @posts %>                        <%# コレクション（_post を各要素に適用） %>
<%= render partial: "post", collection: @posts, as: :post %>
```
- `render @posts` は規約で `_post.html.erb` を探し、ローカル変数 `post` を渡す。

## layout の基本
```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <title>MyApp</title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag "application", media: "all" %>
    <%= javascript_include_tag "application" %>   <%# Sprockets。Turbolinks も含まれる %>
  </head>
  <body>
    <%= render "shared/header" %>
    <%= yield %>                                  <%# ← ここに各ビューが差し込まれる %>
  </body>
</html>
```
- コントローラ単位で `layout "admin"` のように切り替えられる。

## 実務での使い方・定番パターン
- **フォーム共通化**：`new`/`edit` で同じ `_form.html.erb` を `render` する（scaffold もこの形）。
- **`shared/` ディレクトリ**にヘッダ・フッタ・フラッシュ表示などの共通 partial を置く。→ [session_cookie_flash.md](./session_cookie_flash.md)
- **`content_for` / `yield(:name)`** でページ別にタイトルや追加JSを差し込む。
- **コレクションレンダリング**（`render @posts`）は1回のクエリ結果に対して partial を回すだけ。中で関連を辿ると N+1 になるので `includes` 済みで渡す。→ [active_record.md](./active_record.md)
- **SJR（`*.js.erb`）から partial を再利用**：Rails 5 では Ajax 応答で `render partial:` を `j`（escape_javascript）して返す形がまだ使われる。→ [javascript.md](./javascript.md)

## ハマりどころ / アンチパターン
- **partial 内で重い処理/クエリ**：コレクション分だけ走り N+1。データは呼び出し側で用意。
- **ローカル変数とインスタンス変数の混在**：partial は `@変数` に依存させず、`locals` で明示的に渡すと再利用性が上がる。
- **partial の `_` 忘れ**：ファイル名は必ず `_` 始まり。呼び出しは `_` を付けない（`render "post"`）。
- **layout の二重読み込み**：`yield` の中でさらにレイアウトを描画しようとして崩れる。

## 関連
[view.md](./view.md) / [helper.md](./helper.md) / [javascript.md](./javascript.md)
