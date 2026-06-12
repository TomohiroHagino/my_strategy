# アセット（Assets）（Rails 7）

## ひとことで言うと
JavaScript・CSS・画像といった**フロント資産をどう取り込み、どう配信するか**の仕組み。Rails 7では既定で **importmap**（Node不要でJSのESMを使う）＋ **Propshaft**（資産配信）という構成になった。

## 役割・なぜ必要か
- ブラウザに届けるJS/CSSを「どこから読み、どう束ね、どうキャッシュさせるか」を管理するため。
- Rails 7は**「NodeなしでもモダンJSを書ける」**ことを重視し、ビルド工程の重さを避けるため importmap を既定にした（Webpackerは7では既定ではない）。
- 配信側はフィンガープリント付きファイル名で**キャッシュバスティング**し、更新を確実に反映させる。

## 基本の書き方（コード）
```ruby
# config/importmap.rb（Nodeなしでパッケージをpin）
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "stimulus-controllers", to: "controllers/index.js"
```
```erb
<%# app/views/layouts/application.html.erb %>
<%= javascript_importmap_tags %>   <%# importmap利用 %>
<%= stylesheet_link_tag "application" %>
```
```bash
# importmapでパッケージ追加（CDNからpin行を生成）
bin/importmap pin react
# 本番ビルド（資産のプリコンパイル）
RAILS_ENV=production rails assets:precompile
```

## 実務での使い方・定番パターン
- **importmap 既定**：小〜中規模やライブラリ依存が軽い場合に最適。Node/ビルド不要で `pin` だけ管理。
- **バンドルが必要なら gem を追加**：
  - **jsbundling-rails**（esbuild / rollup / webpack を選択）でJSをバンドル。
  - **cssbundling-rails**（Tailwind / Bootstrap / Sass など）でCSSをビルド。
  - これらは `bin/dev`（Procfile.dev + foreman）で `watch` しながら開発するのが定番。
- **Propshaft**（Rails 7の新既定）：資産にダイジェストを付けてそのまま配信する軽量パイプライン。SCSSコンパイル等の変換はしない（変換は jsbundling/cssbundling 側で行う）。
- **フィンガープリント**：`application-<hash>.js` のようにハッシュが付き、内容が変わればURLも変わる＝キャッシュを確実に更新。
- 本番は事前に `assets:precompile` 済みを配る（CDN前段に置くことも多い）。

## ハマりどころ / アンチパターン
- **本番で precompile 忘れ**：資産404や古いまま。デプロイ手順に `assets:precompile` を組み込む。
- **importmap の pin 管理**：CDN由来のバージョン固定がズレる／preload漏れで初回遅延。`bin/importmap audit` で確認。
- **Sprockets資産との混在**：旧 Sprockets と Propshaft を混ぜると挙動が読みづらい。7新規は Propshaft 前提で統一。
- **`config.assets` 系の旧設定の引きずり**（Sprockets時代の記述）→ Propshaftでは不要・無効なものがある。
- importmapはトランスパイルしない＝**素のESMが要る**（JSXやTSはそのままでは不可。必要なら jsbundling）。

## 関連
[hotwire.md](./hotwire.md)
