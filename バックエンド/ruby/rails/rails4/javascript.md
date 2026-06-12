# JavaScript の書き方（Rails 4）

## ひとことで言うと
Rails 4 のフロントは **jQuery 中心**。`jquery_ujs`（jQueryベースのUJS）で `data-*` 属性を使い、Ajax 部分更新は **`*.js.erb`（SJR：Server-generated JavaScript Response）**、ページ遷移高速化は **Turbolinks Classic（3系）** で行う。Hotwire / Turbo / Stimulus は無い。

## 前提（Rails 4 のフロント構成）
- `app/assets/javascripts/application.js` に `//= require jquery` `//= require jquery_ujs` `//= require turbolinks` を並べる（Sprockets）。→ [assets.md](./assets.md)
- `jquery_ujs` が `data-remote` / `data-method` / `data-confirm` を解釈してAjax・DELETE・確認ダイアログを実現する。
- JSは自分で書く（7系のように「ほぼ書かない」ではない）。

## 全体の対応表（Rails 4 のやり方）

| やること | Rails 4 の書き方 |
|---|---|
| 確認ダイアログ・DELETE送信 | `jquery_ujs`（`data: { method: :delete, confirm: "..." }`） |
| Ajaxでフォーム送信 | `remote: true` ＋ `*.js.erb`（SJR） |
| 一覧に行を追加（部分更新） | `create.js.erb` で `$("#posts").append(...)` |
| ページ遷移を速く | Turbolinks Classic（`//= require turbolinks`） |
| 小さな動き（開閉等） | jQuery を直書き（`$(document).on(...)`） |
| ライブラリ追加 | `vendor/assets` に置いて `//= require`（yarn/npm無し） |

---

## ① 削除リンク（確認 → DELETE 送信）

```erb
<%= link_to "削除", post_path(@post),
      method: :delete,
      data: { confirm: "削除しますか？" } %>
```
→ `jquery_ujs` が `data-method="delete"` `data-confirm` を読み、確認ダイアログ→DELETEリクエストを送る。生成HTMLは `<a data-method="delete" data-confirm="...">`。
（7系では `turbo_method` / `turbo_confirm`。Rails 4 は `method` / `confirm`。）

---

## ② Ajaxでフォーム送信して一覧に行を追加（SJR）

サーバが「JSコード」を返して画面を書き換えるのが Rails 4 の定番。

```erb
<%# new フォームを remote 化 %>
<%= form_for @post, remote: true do |f| %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>

<ul id="posts">
  <%= render @posts %>
</ul>
```
```ruby
# posts_controller.rb
def create
  @post = Post.create(post_params)
  respond_to do |format|
    format.js              # → create.js.erb が実行される
    format.html { redirect_to posts_path }
  end
end
```
```erb
<%# app/views/posts/create.js.erb（中身はJavaScript） %>
$("#posts").append("<%= j render(@post) %>");   <%# j = escape_javascript %>
$("#new_post")[0].reset();
```
→ サーバが返したJSがそのままブラウザで実行される。`j`（`escape_javascript`）でHTMLをJS文字列に安全に埋め込む。

---

## ③ ページ遷移を速くする（Turbolinks Classic）

```js
// application.js
//= require turbolinks
```
→ リンククリックを Ajax 化し `<body>` だけ差し替える。`<head>` のCSS/JSは再読込しないので体感が速い。
（7系の Turbo Drive の前身。Rails 4 のは「Turbolinks Classic（3系）」で、`turbolinks:load` ではなく `page:load` イベント。）

**Turbolinks で自前JSが動かない時**
```js
// DOMContentLoaded だと初回しか動かない（遷移後に再評価されないため）
$(document).on("page:load", function () {   // Turbolinks Classic は page:load
  initWidgets();
});
$(document).ready(initWidgets);              // 初回ロード用に両方仕掛ける
```
- イベント名は Turbolinks Classic 系で `page:load` / `page:change` / `page:before-unload`。
- 不具合が多いため `--skip-turbolinks` で外す現場も多い。

---

## ④ ボタンで開閉する（小さな動き＝jQuery 直書き）

```js
// app/assets/javascripts/posts.coffee もしくは .js
$(document).on("click", "#toggle-btn", function () {
  $("#panel").toggle();
});
```
- `$(document).on("click", セレクタ, ...)`（イベント委譲）で書くと、Turbolinks で差し替わった要素にも効く。`$("#toggle-btn").on(...)` の直バインドは遷移後に外れることがある。
（7系は Stimulus で宣言的に書くが、Rails 4 にはStimulusは無い。）

---

## ⑤ 外部ライブラリを追加する

```js
// 1) vendor/assets/javascripts/ に lib.js を置く
// 2) application.js で読む
//= require lib
```
- yarn / npm / Webpacker は無い。gem 化されたライブラリ（`bootstrap-sass` 等）を Gemfile に入れて `//= require` する方法もよく使う。→ [assets.md](./assets.md)

---

## つまずきやすい所（Rails 4 特有）
1. **Turbolinks Classic で自前JSが初回しか動かない** → `$(document).ready` ではなく **`page:load`** も仕掛ける（または Turbolinks を外す）。
2. **遷移後に直バインドのクリックが効かない** → `$(document).on("click", セレクタ, fn)` のイベント委譲で書く。
3. **`*.js.erb` が実行されない** → コントローラで `respond_to { |f| f.js }`、リンク/フォームに `remote: true`、リクエストが `.js` 形式になっているか確認。
4. **削除リンクが反応しない** → `//= require jquery_ujs` が読み込まれているか、`data: { method: :delete }` になっているか確認。
5. **`escape_javascript`（`j`）忘れ** → `*.js.erb` でHTMLを直接埋めると引用符/改行で壊れる。`j render(...)` を使う。

## 関連
[assets.md](./assets.md)（Sprockets） / [view.md](./view.md) / [routing.md](./routing.md)
