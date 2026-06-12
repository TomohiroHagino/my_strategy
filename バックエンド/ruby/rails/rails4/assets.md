# アセット（Asset Pipeline / Sprockets）（Rails 4）

## ひとことで言うと
JavaScript / CSS / 画像を**結合・圧縮・フィンガープリント付与して配信する仕組み**。Rails 4 は **Sprockets のみ**で、`app/assets/` 配下に置き、CoffeeScript / Sass をコンパイルできる（Webpacker は無い）。

## 役割・なぜ必要か
- 複数のJS/CSSファイルを1本に結合してリクエスト数を減らし、圧縮（minify）し、内容ハッシュ付きファイル名（フィンガープリント）でキャッシュを効かせるためにある。
- CoffeeScript→JS、Sass→CSS のプリプロセスもここで行う。

## ディレクトリ構成
```
app/assets/
  javascripts/
    application.js        # マニフェスト（require で他を束ねる）
    posts.coffee          # CoffeeScript も書ける
  stylesheets/
    application.css       # マニフェスト
    posts.scss            # Sass/SCSS
  images/
lib/assets/  vendor/assets/   # 自作ライブラリ/サードパーティ
```

## 基本の書き方（コード）

### マニフェスト（結合の指定）
```js
// app/assets/javascripts/application.js
//= require jquery
//= require jquery_ujs        // jQueryベースUJS（data-remote/method/confirm）
//= require turbolinks        // Turbolinks Classic（外すなら削除）
//= require_tree .            // このディレクトリ配下を全部読む
```
```css
/* app/assets/stylesheets/application.css */
/*
 *= require_self
 *= require_tree .
 */
```

### ビューからの読み込み
```erb
<%= javascript_include_tag "application" %>
<%= stylesheet_link_tag   "application", media: "all" %>
<%= image_tag "logo.png" %>                  <%# /assets/logo-<hash>.png を解決 %>
```
- Sass 内では `image-url("logo.png")` / `asset-path` を使うとフィンガープリント付きURLになる。

## 本番のプリコンパイル
```bash
RAILS_ENV=production rake assets:precompile
```
- 本番は事前コンパイルした `public/assets/` を配信。`config.assets.compile = false`（本番既定）だと未コンパイルのアセットは404になる。
- 個別ファイルを直接 `javascript_include_tag` する場合は `config.assets.precompile` に追加が必要（4ではマニフェスト外のファイルは自動で含まれない）。

## 実務での使い方・定番パターン
- **JSランタイムが必要**：CoffeeScript / uglifier のために Node か `therubyracer` gem を入れる。→ [getting_started.md](./getting_started.md)
- **jQuery プラグイン追加**は `vendor/assets/javascripts/` に置いて `//= require` で読む（Webpacker/yarn は無い）。→ [javascript.md](./javascript.md)
- **CSSフレームワーク**は `bootstrap-sass` 等の gem を Gemfile に入れて `@import` する。
- **Turbolinks Classic** を使うなら `//= require turbolinks`。事故が多いので外す現場もある。→ [javascript.md](./javascript.md)
- 環境ごとに `config.assets.compile` / `config.assets.debug` を確認。

## ハマりどころ / アンチパターン
- **本番でアセット404**：`assets:precompile` を忘れる/`precompile` リストにファイルが入っていない。マニフェスト経由か `precompile` 追加で解決。
- **`config.assets.compile = true` を本番で有効化**：未コンパイルを都度ビルドし激重。本番は precompile した静的ファイル配信。
- **`image_tag` で直パス書き**：`<img src="/assets/logo.png">` と直書きするとフィンガープリントが効かずキャッシュバスティングできない。ヘルパーを使う。
- **JSランタイム不在**：`ExecJS::RuntimeUnavailable`。`therubyracer` か Node を入れる。
- **`require_tree .` の読み込み順依存**：依存順がある JS をツリー一括に任せると順序事故。明示 `require` する。
- （Webpacker / importmap を探すと無い。Rails 4 は Sprockets のみ。Webpacker は 5.1〜、importmap は 7。）

## 関連
[javascript.md](./javascript.md) / [view.md](./view.md) / [getting_started.md](./getting_started.md)
