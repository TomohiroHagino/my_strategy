# Action Text（Rails 6）

## ひとことで言うと
Rails 6 標準のリッチテキスト機能。Trix エディタでの編集と、本文（HTML）の保存・表示を `has_rich_text` で提供する。画像埋め込みは Active Storage 経由。

## 役割・なぜ必要か
- 太字・リンク・箇条書き・画像を含む本文を、自前の WYSIWYG 実装なしで扱える。
- 本文は別テーブル `action_text_rich_texts`（`ActionText::RichText`）に保存され、モデル本体のカラムは増やさない。
- 画像の添付・保存は Active Storage に委譲するため、Active Storage のセットアップが前提。

## 基本の書き方（コード）
```bash
# 事前に Active Storage が必要
bin/rails active_storage:install
bin/rails action_text:install   # マイグレーション生成 + yarn add trix @rails/actiontext
bin/rails db:migrate
```
```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_rich_text :content   # content カラムは作らない
end
```
```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with model: @post do |form| %>
  <%= form.label :content %>
  <%= form.rich_text_area :content %>
  <%= form.submit %>
<% end %>
```
```erb
<%# 表示（HTML はサニタイズ済みで出力される） %>
<%= @post.content %>
```
```ruby
# Strong Parameters
def post_params
  params.require(:post).permit(:title, :content)
end
```
```js
// app/javascript/packs/application.js（Webpacker）
require("trix")
require("@rails/actiontext")
```

## 実務での使い方・定番パターン
- ブログ記事・お知らせ・コメントなど、整形が必要な本文に使う。
- 画像をエディタにドラッグすると Active Storage に保存され、本文に埋め込まれる。
- N+1 を避けるため一覧取得時は `Post.with_rich_text_content` を使う。

## ハマりどころ / アンチパターン
- Active Storage 未設定だと画像埋め込みが失敗する。先に `active_storage:install` を実行する。
- `has_rich_text :content` に対応するカラムを users テーブルに作らない。データは `ActionText::RichText` の別テーブル。
- `<%= @post.content %>` は HTML を出力する。これは Action Text 側でサニタイズされるが、信頼できない HTML を別経路で混ぜない。
- Webpacker / yarn での `trix` `@rails/actiontext` 依存が必要。`application.js` に require を入れないとエディタが表示されない。

## 関連
[active_storage.md](./active_storage.md) / [webpacker.md](./webpacker.md) / [model.md](./model.md) / [view.md](./view.md) / [strong_parameters.md](./strong_parameters.md) / [javascript.md](./javascript.md) / [pitfalls.md](./pitfalls.md)
