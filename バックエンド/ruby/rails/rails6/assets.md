# アセット管理（Sprockets + Webpacker の役割分担）（Rails 6）

## ひとことで言うと
Rails 6 のフロント資産は二系統に分かれる。CSS・画像・フォントは Sprockets（アセットパイプライン、`app/assets/`）、JavaScript（npm パッケージ）は Webpacker（`app/javascript/`）が既定で担当する。

## 役割・なぜ必要か
- Sprockets は CSS や画像を連結・ミニファイし、内容ハッシュ（フィンガープリント）付きのファイル名で `public/assets/` に出力する。フィンガープリントによりキャッシュを安全に長期化できる。
- Webpacker は JS の ES Modules / npm パッケージを束ねて `public/packs/` に出力する。
- 二系統あるのは Rails 6 が「JS は webpack に任せ、CSS・画像は従来どおり Sprockets で扱う」という移行期の構成を取ったため。どちらに何を置くかを間違えると本番で 404 になる。

## 役割分担（既定）

| 資産 | 担当 | 置き場所 | ビューのヘルパ |
|---|---|---|---|
| JavaScript（npm 資産含む） | Webpacker | `app/javascript/packs/` | `javascript_pack_tag` |
| CSS | Sprockets（既定） | `app/assets/stylesheets/` | `stylesheet_link_tag` |
| 画像 | Sprockets | `app/assets/images/` | `image_tag` / `asset_path` |
| フォント | Sprockets | `app/assets/fonts/` | `asset_path` |

（CSS は Webpacker に寄せる構成も可能。その場合は `stylesheet_pack_tag` を使う。既定は Sprockets。）

## 基本の書き方（コード）

ビュー（`app/views/layouts/application.html.erb`）：
```erb
<%= stylesheet_link_tag 'application', media: 'all' %>   <%# Sprockets: app/assets/stylesheets/application.css %>
<%= javascript_pack_tag 'application' %>                 <%# Webpacker: app/javascript/packs/application.js %>
```

画像の参照（Sprockets。フィンガープリント付き URL を生成）：
```erb
<%= image_tag "logo.png" %>            <%# app/assets/images/logo.png %>
<%= image_tag asset_path("logo.png") %>
```

CSS 内から画像を参照（`*.css` を `*.scss`/`*.erb` ではなく素の `.css` にすると `asset_path` は使えないので注意）：
```scss
/* app/assets/stylesheets/application.scss */
.logo { background-image: image-url("logo.png"); }
```

precompile 対象を明示するマニフェスト（`app/assets/config/manifest.js`）：
```js
//= link_tree ../images
//= link_directory ../stylesheets .css
```

## 実務での使い方・定番パターン

本番ビルド（Sprockets と Webpacker の両方をコンパイルする）：
```bash
RAILS_ENV=production bin/rails assets:precompile
```
出力先：
- Sprockets → `public/assets/`
- Webpacker → `public/packs/`

CDN を使う場合は `config.asset_host` を設定する（`asset_path` 系が CDN の URL を返すようになる）：
```ruby
# config/environments/production.rb
config.asset_host = "https://cdn.example.com"
```

precompile 対象に独自ファイルを足したいとき（既定で拾われないファイル）：
```js
// app/assets/config/manifest.js
//= link admin.css
//= link print.css
```

## ハマりどころ / アンチパターン
- 本番で CSS や画像が 404：precompile 対象から漏れている。`app/assets/config/manifest.js` に `//= link ...` を追記する。`application.css`/`application.js` から `require`/`import` で辿れないファイルは個別に link が必要。
- `asset_path("foo.png")` で `Asset ... was not found` になる：ファイル名・拡張子・置き場所（`app/assets/images/`）を確認する。precompile 後は `public/assets/foo-<hash>.png` を参照するため、手書きの `/assets/foo.png` は壊れる。常にヘルパ経由で参照する。
- JS の置き場所間違い：JS を `app/assets/javascripts/` に置いても Rails 6 既定では読まれない。JS は `app/javascript/packs/` + `javascript_pack_tag`（Webpacker）。
- `stylesheet_link_tag`（Sprockets）と `stylesheet_pack_tag`（Webpacker）の取り違え。CSS を Sprockets で管理しているなら `stylesheet_link_tag`。
- `config.assets.compile = true` を本番で有効にすると、初回アクセス時にオンザフライでコンパイルが走り遅く・不安定になる。本番は precompile 済み前提（`false`）にする。
- フィンガープリントが変わらず古い CSS が表示される：ブラウザ/CDN キャッシュ。デプロイ時に `public/assets`/`public/packs` を作り直し、CDN をパージする。

## 関連
[webpacker.md](./webpacker.md) / [javascript.md](./javascript.md) / [view.md](./view.md) / [active_storage.md](./active_storage.md) / [pitfalls.md](./pitfalls.md)
