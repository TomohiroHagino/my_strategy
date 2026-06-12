# JavaScript の書き方（Rails 5）

## ひとことで言うと
Rails 5 のフロントは **Turbolinks 5（ページ遷移の高速化）＋ rails-ujs（リンク/フォームの拡張）** が中心。Ajax更新は **`*.js.erb`（SJR）＋ jQuery**、外部ライブラリは **Sprockets か Webpacker（5.1〜）** で入れる。Hotwire（Turbo/Stimulus）は無い。
このページは、よくあるタスクを **タスク別の具体コード**で並べる。

## 全体像（Rails 5 のフロント構成）

| やること | Rails 5 のやり方 |
|---|---|
| ページ遷移を速く | Turbolinks 5（自動。`<body>`差し替え） |
| 削除リンク・確認ダイアログ | rails-ujs（`method: :delete` / `data-confirm`） |
| 部分更新（行追加など） | `*.js.erb`（SJR）＋ jQuery |
| 小さな動き（開閉等） | jQuery を直書き |
| ライブラリ追加 | Sprockets（`//= require`）/ Webpacker（yarn・5.1〜） |

> Rails 7 との違い：7 は Turbo/Stimulus/importmap。Rails 5 は **Turbolinks + rails-ujs（jQuery系）** で、属性は `turbo_method` ではなく `method` を使う。

---

## ① 削除リンク（確認 → DELETE 送信）

```erb
<%= link_to "削除", post_path(@post),
      method: :delete,
      data: { confirm: "削除しますか？" } %>
```
→ **rails-ujs**（5.1で jquery_ujs から置換されたバニラJS）が `data-method` `data-confirm` を読んで、確認ダイアログ＋DELETE送信を行う。JSは自分で書かない。
- レイアウトに `//= require rails_ujs`（5.1+）/ `//= require jquery_ujs`（5.0）が必要。

---

## ② 一覧に行を追加（フォーム送信で部分更新 / SJR）

サーバが「JSコード」を返して画面を書き換える（SJR = Server-generated JavaScript Responses）。

```erb
<%# リンク/フォームを remote: true にする → Ajax 送信になる %>
<%= form_with model: @post, remote: true do |f| %>   <%# 5.1+。5.0 は form_for ..., remote: true %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>
```
```ruby
# posts_controller.rb
def create
  @post = Post.create!(post_params)
  respond_to do |format|
    format.js                       # → create.js.erb が実行される
    format.html { redirect_to posts_path }
  end
end
```
```erb
<%# app/views/posts/create.js.erb（中身はブラウザで実行されるJS） %>
$("#posts").append("<%= j render(@post) %>");   <%# j = escape_javascript %>
$("#new_post")[0].reset();
```
→ サーバが返したJS文字列をブラウザがそのまま実行してDOMを書き換える。Rails 5 ではこの SJR が現役。

---

## ③ ページ遷移を速くする（Turbolinks 5）

**Turbolinks 5** がレイアウトに読み込まれていれば、リンク・フォームが自動でAjax化され `<body>` だけ差し替わる（フルリロードしない）。**設定もコードも基本不要**。

```javascript
// app/assets/javascripts/application.js
//= require turbolinks
```
- 注意：通常の `DOMContentLoaded` は Turbolinks 遷移では発火しない。イベントは **`turbolinks:load`** を使う。
  ```js
  document.addEventListener("turbolinks:load", function () {
    // ページ遷移ごとに毎回走る初期化
  });
  ```

---

## ④ ボタンで開閉する（小さな動き / jQuery 直書き）

```javascript
// app/assets/javascripts/posts.js（Sprockets で読み込まれる）
$(document).on("turbolinks:load", function () {     // ★ ready でなく turbolinks:load
  $("#toggle-btn").on("click", function () {
    $("#panel").toggle();
  });
});
```
→ Rails 5 には Stimulus が無いので、小さな動きは jQuery を直書きする。イベント登録は `turbolinks:load` に乗せる（`$(document).ready` だと2ページ目以降で動かない）。

---

## ⑤ 外部ライブラリを追加する

**Sprockets（既定）**
```javascript
// vendor に置く / gem で入れる場合
//= require lodash
```

**Webpacker（5.1〜・yarn）**
```bash
yarn add lodash          # Node / yarn が必要
```
```javascript
// app/javascript/packs/application.js
import _ from "lodash"
```
```erb
<%= javascript_pack_tag "application" %>   <%# Sprockets の include_tag とは別 %>
```

---

## つまずきやすい所（Rails 5 特有）
1. **2ページ目以降でJSが動かない** → `$(document).ready` でなく **`turbolinks:load`** を使う（Turbolinks がフルリロードしないため）。
2. **削除リンクが反応しない** → rails-ujs（5.1+）/ jquery_ujs（5.0）がレイアウトで読み込まれていない。`//= require` を確認。
3. **`form_with` が勝手にAjaxになる（5.1）** → 5.1 の `form_with` は既定でリモート。通常のHTML送信なら `local: true`、SJRしたいなら `remote: true`（明示）。
4. **`turbolinks:load` が二重発火** → スクリプトを `<head>` で複数回読むと二重バインドになる。`require` の重複に注意。

## 関連
[assets.md](./assets.md)（Sprockets / Webpacker） / [view.md](./view.md) / [partial_layout.md](./partial_layout.md)
