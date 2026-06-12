# JavaScript の書き方（Rails 7）

## ひとことで言うと
Rails 7 は、昔の「jQueryやJSを自分で書く」スタイルから、「**Turbo + Stimulus でほぼJSを書かない**」スタイルに変わった。
このページは、よくあるタスクを **「過去の書き方 → 今の書き方」** で具体的に並べる。

## 全体の対応表（過去 → 今）

| やること | 過去（Rails 6以前） | 今（Rails 7） |
|---|---|---|
| ページ遷移を速く | Turbolinks / 自前Ajax | Turbo Drive（自動） |
| 部分更新（行追加など） | `*.js.erb`（SJR）+ jQuery | Turbo Streams |
| 一部だけ差し替え | 自前Ajax + innerHTML | Turbo Frames |
| 確認ダイアログ・DELETE | rails-ujs（`data-confirm` 等） | `turbo_confirm` / `turbo_method` |
| 小さな動き（開閉等） | jQuery を直書き | Stimulus |
| ライブラリ追加 | Webpacker（yarn / npm） | importmap |

---

## ① 削除リンク（確認 → DELETE 送信）

**過去（rails-ujs）**
```erb
<%= link_to "削除", post_path(@post),
      method: :delete,
      data: { confirm: "削除しますか？" } %>
```
→ `rails-ujs` というJSが `data-method` `data-confirm` を読んで動いていた。

**今（Turbo）**
```erb
<%= link_to "削除", post_path(@post),
      data: { turbo_method: :delete, turbo_confirm: "削除しますか？" } %>
```
→ 属性名が `turbo_` 付きに変わっただけ。JSは自分で書かない。

---

## ② 一覧に行を追加（フォーム送信で部分更新）

**過去（`*.js.erb`＝SJR ＋ jQuery）** … サーバが「JSコード」を返して画面を書き換える
```ruby
# posts_controller.rb
def create
  @post = Post.create!(post_params)
  # → create.js.erb が実行される
end
```
```js
<%# create.js.erb %>
$("#posts").append("<%= j render(@post) %>");
$("#new_post")[0].reset();
```

**今（Turbo Streams）** … サーバが「HTML＋操作指示」を返す
```ruby
# posts_controller.rb
def create
  @post = Post.create!(post_params)
  respond_to do |format|
    format.turbo_stream            # → create.turbo_stream.erb
    format.html { redirect_to posts_path }
  end
end
```
```erb
<%# create.turbo_stream.erb（中身はHTML） %>
<%= turbo_stream.append "posts", @post %>
```
→ JSを書かず「`posts` に `@post` を追加して」とHTMLで指示する。

---

## ③ ページ遷移を速くする

**過去**：`Turbolinks`（Rails 5）や自前Ajax。`$(document).on("turbolinks:load", ...)` などを書いていた。

**今**：**Turbo Drive** が標準で効く。**コードも設定も不要**。リンク・フォームが自動でAjax化され、`<body>` だけ差し替わる。

→ 詳しくは [turbo_drive.md](./turbo_drive.md)

---

## ④ ボタンで開閉する（小さな動き）

**過去（jQuery 直書き）**
```js
// app/assets/javascripts/application.js など
$(document).on("click", "#toggle-btn", function () {
  $("#panel").toggle();
});
```
→ どのHTMLに効くJSか、コードを追わないと分からない。

**今（Stimulus）**
```erb
<div data-controller="toggle">
  <button data-action="click->toggle#switch">開閉</button>
  <div data-toggle-target="panel">中身</div>
</div>
```
```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"
export default class extends Controller {
  static targets = ["panel"]
  switch() { this.panelTarget.classList.toggle("hidden") }
}
```
→ HTMLを見れば「toggle という動きが付いている」と分かる。グローバルに `$(...)` を撒かない。

---

## ⑤ 外部ライブラリを追加する

**過去（Webpacker）**
```bash
yarn add lodash          # Node / npm が必要
```
```js
import _ from "lodash"
```

**今（importmap）**
```bash
bin/importmap pin lodash # Node / npm 不要
```
```js
import _ from "lodash"   // 使い方は同じ
```

---

## つまずきやすい所（Rails 7 特有）
1. **Turbo でページ移動すると自前JSが動かない** → `DOMContentLoaded` ではなく **`turbo:load`** を使う
2. **フォームのエラーが画面に出ない** → コントローラで **`status: :unprocessable_entity`（422）** を返す
3. **削除リンクが反応しない** → `method: :delete` ではなく **`turbo_method: :delete`**（Rails 7）

## 関連
[hotwire.md](./hotwire.md)（Turbo / Stimulus の詳細） / [assets.md](./assets.md)（importmap） / [view.md](./view.md)
