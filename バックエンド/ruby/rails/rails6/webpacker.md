# Webpacker（Rails 6）

## ひとことで言うと
Webpacker は、JavaScript バンドラの webpack を Rails から使えるようにするラッパー。Rails 6 では `rails new` で既定導入され、アプリの JS は `app/javascript/packs/application.js` を起点に書き、依存は `yarn add` で追加する。

## 役割・なぜ必要か
- Rails 6 のアプリ JS（npm パッケージを含む）は Webpacker が担当する。CSS・画像・フォントは Sprockets（アセットパイプライン）が担当する、という役割分担が既定。
- `import` / `export` の ES Modules、npm パッケージ（React, Vue, lodash 等）、Babel/トランスパイル、ソースマップ、本番のミニファイをまとめて扱える。
- 旧 Rails（5.x まで）の `app/assets/javascripts/application.js` + Sprockets での JS 管理を置き換える位置づけ。

## 基本の書き方（コード）

エントリは `app/javascript/packs/application.js`：
```js
// app/javascript/packs/application.js
import Rails from "@rails/ujs"
import Turbolinks from "turbolinks"
import "../channels"   // Action Cable を使う場合

Rails.start()
Turbolinks.start()
```

ビューでの読み込み（`app/views/layouts/application.html.erb`）：
```erb
<%= javascript_pack_tag 'application' %>
<%= stylesheet_pack_tag 'application' %>   <%# Webpacker でCSSも扱う構成のとき %>
```
`javascript_pack_tag` は Webpacker が出力した `public/packs/` のファイルを参照する。Sprockets の `javascript_include_tag`（`public/assets/` を参照）とは別物。混同しない。

パッケージ追加：
```bash
yarn add lodash
```
```js
// app/javascript/packs/application.js など
import _ from "lodash"
console.log(_.chunk([1, 2, 3, 4], 2))
```

## 実務での使い方・定番パターン

開発・ビルドのコマンド：
```bash
bin/webpack-dev-server   # ホットリロード付き開発サーバ（rails server と別プロセスで起動）
bin/webpack              # 一度だけビルド（dev-server を使わない場合）
```
本番では `assets:precompile` が内部で `webpacker:compile` を呼ぶため、デプロイ時に明示的に webpack を叩く必要はない：
```bash
RAILS_ENV=production bin/rails assets:precompile
```

設定ファイル：
- `config/webpacker.yml`：エントリのパス、出力先、`compile`（リクエスト時の自動コンパイル可否）、`extract_css` などの環境別設定。
- `config/webpack/environment.js`：webpack の plugin/loader をカスタマイズする入口。

フレームワーク・ツールの後付けは専用タスクを使う：
```bash
rails webpacker:install:stimulus   # Stimulus を追加（app/javascript/controllers/ が生成される）
rails webpacker:install:react      # React
rails webpacker:install:vue        # Vue
```

jQuery を Webpacker で使い、プラグインが `$`/`jQuery` をグローバル前提で参照する場合は `ProvidePlugin` で注入する：
```js
// config/webpack/environment.js
const { environment } = require('@rails/webpacker')
const webpack = require('webpack')

environment.plugins.prepend('Provide',
  new webpack.ProvidePlugin({
    $: 'jquery',
    jQuery: 'jquery'
  })
)
module.exports = environment
```

CSS を Sprockets ではなく Webpacker に寄せる構成も可能。その場合は `packs/application.js` 内で CSS を `import` し、ビューで `stylesheet_pack_tag` を使う：
```js
// app/javascript/packs/application.js
import "../stylesheets/application.css"
```

## ハマりどころ / アンチパターン
- `javascript_include_tag 'application'`（Sprockets）と `javascript_pack_tag 'application'`（Webpacker）の混同。Webpacker の JS を読みたいなら `javascript_pack_tag`。両方書くと二重読み込みやエラーになる。
- `node_modules` 未インストールや yarn 未導入の環境でビルドが失敗する。Webpacker は Node.js と yarn を必須とする。
- 本番の `assets:precompile` で webpack が大量のメモリを使い、メモリ不足（OOM）でデプロイが落ちる。事前ビルド・swap・`NODE_OPTIONS=--max-old-space-size` 等で対処する。
- Turbolinks 環境では、`DOMContentLoaded` で初期化したコードが 2 回目以降のページ遷移で発火しない。`turbolinks:load` を使う：
  ```js
  document.addEventListener("turbolinks:load", () => { /* 初期化 */ })
  ```
- `packs/application.js` に書いた変数・関数はモジュールスコープであり、グローバルではない。`<script>` インラインや HTML の `onclick` から呼べると誤解しない。グローバルに公開したいなら `window.foo = foo` と明示するか、Stimulus に移す。
- `config/webpacker.yml` の `compile: true` に頼ると開発でリクエストごとにビルドが走り遅い。`bin/webpack-dev-server` を併用する。

> Rails 7 では Webpacker を卒業し、Node 不要の importmap（または jsbundling/esbuild）が既定になった。

## 関連
[assets.md](./assets.md) / [javascript.md](./javascript.md) / [getting_started.md](./getting_started.md) / [view.md](./view.md) / [pitfalls.md](./pitfalls.md)
