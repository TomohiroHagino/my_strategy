# Hotwire（Turbo / Stimulus）（Rails 7）

## ひとことで言うと
**JavaScriptをほとんど書かずに、SPA風の速い画面更新を実現するRails標準のフロント手法**。Rails 7 から既定で組み込まれる。`Turbo` と `Stimulus` の2本柱（＋Turbo Streams）。

## 役割・なぜ必要か
- 「React等のSPAを組むほどではないが、全ページリロードは古い」という大半のWeb UIを、サーバサイドHTML中心のまま今風にするためにある。
- サーバが返すのは基本HTML。フロントの状態管理やビルドの重さを避けつつ、部分更新・即時反映を得られる。

## 構成要素

### Turbo Drive とは
リンククリック/フォーム送信を**自動でAjax化**し、`<body>` だけ差し替える（フルリロードを避ける）。設定不要で効く。

### Turbo Frames とは
ページの一部を独立した更新単位にする。`<turbo-frame>` 内のリンク/フォームは、そのフレーム内だけを置き換える。
```erb
<turbo-frame id="post_<%= post.id %>">
  <%= post.title %>
  <%= link_to "編集", edit_post_path(post) %>  <%# このフレーム内だけ差し替わる %>
</turbo-frame>
```

### Turbo Streams とは
サーバから「**この要素を append / prepend / replace / update / remove せよ**」とHTML片で指示し、DOMを部分更新する。フォーム応答やActionCable（リアルタイム）で使う。
```erb
<%# create.turbo_stream.erb %>
<%= turbo_stream.append "posts", partial: "post", locals: { post: @post } %>
<%= turbo_stream.update "posts_count", @posts.size %>
```
```ruby
# コントローラ：respond_to で turbo_stream を返す
respond_to do |format|
  format.turbo_stream
  format.html { redirect_to posts_path }
end
```

### Stimulus とは
HTMLに `data-controller` などの属性を付け、**控えめなJSで振る舞いを足す**軽量フレームワーク。状態はDOMに置く発想。
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

## 実務での使い方・定番パターン
- 一覧への行追加・削除、いいね、インライン編集、フラッシュ通知、無限スクロール … は Turbo Frames / Streams で完結。
- リアルタイム（複数ユーザに即反映）は `broadcasts_to` ＋ ActionCable で Turbo Stream を配信。
- JSが必要な細かいUI（モーダル開閉・コピー・ドラッグ）は Stimulus。
- importmap（7既定）でJSはNodeなしに管理。

## ハマりどころ / アンチパターン
- **フォームエラー時は `status: :unprocessable_entity`**（422）を返さないとTurboが再描画しない（7の頻出落とし穴）。→ [controller.md](./controller.md)
- **リダイレクトは 303（`status: :see_other`）** が必要な場面がある（Turboのフォーム送信後）。
- **Turbo Driveでページ遷移してもJSは再評価されない**：`DOMContentLoaded` でなく `turbo:load` を使う。グローバルJSの初期化漏れに注意。
- `turbo-frame` のIDが一致しないと「Content missing」。フレームIDの設計に注意。
- 重い・状態が複雑なUIまでHotwireで頑張りすぎない（そこはReact等を検討）。

## 関連
[view.md](./view.md) / [controller.md](./controller.md)
