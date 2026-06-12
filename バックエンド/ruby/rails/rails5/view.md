# ビュー（View）（Rails 5）

## ひとことで言うと
コントローラが用意したデータ（`@変数`）を使って、**ブラウザに返すHTMLを組み立てるテンプレート層**。`app/views/` に置き、既定は ERB（`.html.erb`）。

## 役割・なぜ必要か
- MVCの「表示」担当。データを画面の見た目に変換する責務だけを持つ（業務ロジックは持たない）。
- アクション名と同名のテンプレート（`index` → `index.html.erb`）が自動で選ばれる規約があり、`render` を書かなくても対応するビューが描画される。

## 基本の書き方（コード）
```erb
<%# app/views/posts/index.html.erb %>
<h1>記事一覧</h1>

<% @posts.each do |post| %>
  <article>
    <h2><%= link_to post.title, post_path(post) %></h2>   <%# = で出力（自動エスケープ） %>
    <p><%= post.body.truncate(100) %></p>
  </article>
<% end %>

<%# Rails 5.1+ は form_with が推奨（5.0 までは form_for / form_tag） %>
<%= form_with(model: @post, local: true) do |f| %>
  <%= f.label :title %>
  <%= f.text_field :title %>
  <%= f.submit "保存" %>
<% end %>
```
- `<%= %>` は値を出力（**自動でHTMLエスケープ**＝XSS対策）、`<% %>` は出力しない（制御構文）。

## 実務での使い方・定番パターン
- **partial（部分テンプレート）** で繰り返し・共通UIを切り出す（`_post.html.erb` を `render @posts`）。→ [partial_layout.md](./partial_layout.md)
- **layout**（`app/views/layouts/application.html.erb`）で共通の枠（ヘッダ・フッタ・`<%= yield %>`）。→ [partial_layout.md](./partial_layout.md)
- **表示整形はヘルパー**へ（日付フォーマット・金額表示など）。ビューにロジックを書かない。→ [helper.md](./helper.md)
- **`form_with`（5.1〜）**：`form_for` / `form_tag` を統合した新しいフォームヘルパー。5.0 までは `form_for`（モデル用）/ `form_tag`（汎用）を使う。
- **Turbolinks 5**：レイアウトの `<head>` に turbolinks が読み込まれ、リンク遷移が自動でAjax化される。→ [javascript.md](./javascript.md)

## ハマりどころ / アンチパターン
- **`<%= %>` と `<% %>` の取り違え**：`<% post.title %>` だと画面に出ない。出力は `=` 付き。
- **`raw` / `html_safe` の乱用**：自動エスケープを外すと XSS の穴になる。ユーザ入力には使わない。→ [security.md](./security.md)
- **ビューにクエリ/業務ロジック**：`@posts = Post.where(...)` をビューで書くと N+1 や責務違反。コントローラ/モデルへ。
- **`form_with` の `local: true` 忘れ**（5.1）：5.1 の `form_with` は既定でリモート（Ajax）送信になり、通常のHTML送信を期待すると挙動が変わる。HTML送信なら `local: true` を付ける。
- **インスタンス変数の渡し忘れ**：コントローラで `@post` を設定し忘れると `nil` で落ちる。

## 関連
[controller.md](./controller.md) / [helper.md](./helper.md) / [partial_layout.md](./partial_layout.md) / [javascript.md](./javascript.md)
