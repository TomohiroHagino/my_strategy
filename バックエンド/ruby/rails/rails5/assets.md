# アセット（Assets）（Rails 5）

## ひとことで言うと
JavaScript・CSS・画像といった**フロント資産をどう取り込み、どう配信するか**の仕組み。Rails 5 の既定は **Sprockets（アセットパイプライン）**。5.1 からは **Webpacker（yarn でJSを管理）** も選べる。

## 役割・なぜ必要か
- ブラウザに届けるJS/CSSを「どこから読み、どう束ね（連結・圧縮）、どうキャッシュさせるか」を管理するため。
- 本番ではファイルをまとめて圧縮し、フィンガープリント（ハッシュ）付きファイル名で配信して**キャッシュバスティング**する。

## Sprockets（既定）の基本
```javascript
// app/assets/javascripts/application.js（manifest。//= require で連結）
//= require rails-ujs        // 5.1+。5.0 は jquery_ujs
//= require turbolinks
//= require_tree .
```
```scss
// app/assets/stylesheets/application.css（同様に *= require で連結）
/*
 *= require_tree .
 *= require_self
 */
```
```erb
<%# レイアウトで読み込む %>
<%= javascript_include_tag "application" %>
<%= stylesheet_link_tag "application" %>
<%= image_tag "logo.png" %>            <%# app/assets/images/ から探す %>
```
```bash
# 本番ビルド（資産のプリコンパイル）
RAILS_ENV=production rails assets:precompile
```

## Webpacker（5.1〜・選択）の基本
```bash
rails new myapp --webpack      # 生成時に導入（yarn 必須）
# 既存に後入れ
rails webpacker:install
```
```javascript
// app/javascript/packs/application.js（Webpacker のエントリ）
import "./hello"
```
```erb
<%# レイアウトで読み込む（Sprockets とは別タグ） %>
<%= javascript_pack_tag "application" %>
```
- **Sprockets と Webpacker は併存**できる（Rails 5 では Sprockets が主、Webpacker は「JSだけモダンに」という補助的な立ち位置が多い）。

## 実務での使い方・定番パターン
- **小〜中規模は Sprockets で十分**：`//= require` で連結し、`assets:precompile` で配る。Node 不要。
- **モダンJS（npmパッケージ・ES Modules）を使いたいとき Webpacker**：React/Vue を載せる、ビルドが要るライブラリを使う場合。
- **フィンガープリント**：`application-<hash>.js` のようにハッシュが付き、内容が変わればURLも変わる＝キャッシュを確実に更新。
- 本番は事前に `assets:precompile` 済みを配る（CDN前段に置くことも多い）。
- 画像・CSS内パスは `image-url` / `asset_path` 系ヘルパーでフィンガープリント対応のパスを出す。

## ハマりどころ / アンチパターン
- **本番で precompile 忘れ**：資産404や古いまま。デプロイ手順に `assets:precompile` を組み込む。
- **`config.assets.compile = true` を本番で頼る**：実行時コンパイルは遅く本番非推奨。事前 precompile が原則。
- **precompile 対象漏れ**：`application.js`/`application.css` 以外を直接読むなら `config.assets.precompile += [...]` に追加が要る。
- **Sprockets と Webpacker の二重管理**：どちらでJSを読んでいるか曖昧になる。タグ（`javascript_include_tag` vs `javascript_pack_tag`）と置き場（`app/assets` vs `app/javascript`）を意識する。
- **Webpacker は yarn 必須**：5.0 には無い。導入したのに yarn が無いとビルドできない。

## 関連
[javascript.md](./javascript.md) / [view.md](./view.md)
