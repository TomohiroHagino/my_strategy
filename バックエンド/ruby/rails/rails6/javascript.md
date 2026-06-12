# JavaScript の書き方（Rails 6）

## ひとことで言うと
Rails 6 の既定は **Turbolinks 5 + rails-ujs + Webpacker**。Hotwire（Turbo/Stimulus を中心とした構成）は標準ではない。このページは、よくあるタスクを「Rails 6 ではこう書く」で具体コードと共に並べる。

## 役割・なぜ必要か
- `rails-ujs`：`data-method`・`data-confirm`・`data-remote` といった HTML 属性を読み取り、DELETE 送信・確認ダイアログ・Ajax フォーム送信を JS を書かずに実現する。`packs/application.js` で `Rails.start()` 済み。
- `Turbolinks 5`：リンク・フォーム送信を Ajax 化し `<body>` だけ差し替えてページ遷移を速くする。`Turbolinks.start()` 済み。
- `Webpacker`：自分で書く JS・npm パッケージの管理。エントリは `app/javascript/packs/application.js`。

## 基本の書き方（コード）

エントリ（既定）：
```js
// app/javascript/packs/application.js
import Rails from "@rails/ujs"
import Turbolinks from "turbolinks"
Rails.start()
Turbolinks.start()
```

---

## ① 削除リンク（確認 → DELETE 送信）

```erb
<%= link_to "削除", post_path(@post),
      method: :delete,
      data: { confirm: "削除しますか？" } %>
```
`rails-ujs` が `data-method="delete"` と `data-confirm` を読み、確認後に DELETE を送る。JS は自分で書かない。

> Rails 7 では `data: { turbo_method: :delete, turbo_confirm: "..." }` に変わる。

---

## ② フォームの remote 送信 + `*.js.erb`（SJR）で部分更新

`form_with` は Rails 6 では既定で remote（Ajax）送信になる。サーバは `*.js.erb`（SJR = Server-generated JavaScript Responses）を返して画面を書き換える。

```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with model: @post do |f| %>   <%# 既定で remote: true（Ajax） %>
  <%= f.text_field :title %>
  <%= f.submit %>
<% end %>
```
```ruby
# app/controllers/posts_controller.rb
def create
  @post = Post.create!(post_params)
  # → create.js.erb が実行される（format.js）
end
```

vanilla JS の SJR（jQuery 不要）：
```erb
<%# app/views/posts/create.js.erb %>
document.getElementById("posts")
  .insertAdjacentHTML("beforeend", "<%= j render(@post) %>");
document.getElementById("new_post").reset();
```

jQuery を使う SJR（`yarn add jquery` + `ProvidePlugin` 済み前提）：
```erb
<%# app/views/posts/create.js.erb %>
$("#posts").append("<%= j render(@post) %>");
$("#new_post")[0].reset();
```

同期送信（リダイレクトしたい）にしたいなら明示的に `local: true`：
```erb
<%= form_with model: @post, local: true do |f| %>
```

> Rails 7 では SJR の代わりに Turbo Streams（`*.turbo_stream.erb`）を使う。

---

## ③ ページ遷移を速くする

Turbolinks 5 が標準で効く。リンク・フォームが自動で Ajax 化され `<body>` が差し替わる。コードは不要。

特定のリンクで無効化したいとき：
```erb
<%= link_to "通常遷移", report_path, data: { turbolinks: false } %>
```
```html
<a href="/full-reload" data-turbolinks="false">通常遷移</a>
```

ページ読み込み時の初期化は `turbolinks:load` で行う（後述のハマりどころ参照）：
```js
document.addEventListener("turbolinks:load", () => {
  console.log("ページ表示・遷移のたびに発火")
})
```

> Rails 7 では Turbolinks の後継 Turbo Drive が標準で、イベントは `turbo:load`。

---

## ④ 小さな DOM 操作（開閉など）

最小構成は pack に直接書く：
```js
// app/javascript/packs/application.js
document.addEventListener("turbolinks:load", () => {
  const btn = document.getElementById("toggle-btn")
  if (!btn) return
  btn.addEventListener("click", () => {
    document.getElementById("panel").classList.toggle("hidden")
  })
})
```

規模が出てきたら Stimulus を後付けする：
```bash
rails webpacker:install:stimulus   # app/javascript/controllers/ が生成される
```
```erb
<div data-controller="toggle">
  <button data-action="click->toggle#switch">開閉</button>
  <div data-target="toggle.panel">中身</div>
</div>
```
```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "stimulus"
export default class extends Controller {
  static targets = ["panel"]
  switch() { this.panelTarget.classList.toggle("hidden") }
}
```

---

## ⑤ 外部ライブラリを追加する

Rails 6 は Webpacker なので yarn + import で追加する：
```bash
yarn add lodash
```
```js
// app/javascript/packs/application.js など
import _ from "lodash"
console.log(_.chunk([1, 2, 3, 4], 2))
```

> Rails 7 は importmap が既定で `bin/importmap pin lodash`（Node 不要）。Rails 6 は importmap ではなく Webpacker。

---

## つまずきやすい所（Rails 6 特有）
1. **2 回目以降のページ遷移で自前 JS が効かない**：`DOMContentLoaded` は Turbolinks の遷移では再発火しない。`turbolinks:load` を使う。
2. **`form_with` が勝手に Ajax になり想定外**：Rails 6 では `form_with` が既定 remote。リダイレクトしたいフォームは `local: true` を付ける。
3. **削除リンクが反応しない / GET になる**：`method: :delete`（rails-ujs）が必要。`Rails.start()` が呼ばれているか、`@rails/ujs` が読み込まれているかを確認する。
4. **jQuery プラグインで `$ is not defined`**：Webpacker の JS はモジュールスコープ。`config/webpack/environment.js` で `ProvidePlugin` を使い `$`/`jQuery` を注入する（[webpacker.md](./webpacker.md) 参照）。
5. **pack に書いた関数が HTML の `onclick` から呼べない**：pack の変数はグローバルでない。`window.foo = foo` で明示公開するか、`addEventListener` / Stimulus に移す。

## 関連
[webpacker.md](./webpacker.md) / [assets.md](./assets.md) / [view.md](./view.md) / [partial_layout.md](./partial_layout.md) / [controller.md](./controller.md) / [pitfalls.md](./pitfalls.md)
