# ビュー（Views）（Sinatra 3/4）

## ひとことで言うと
**HTMLを組み立てる仕組み**。`erb :index` のように「テンプレートを描画する」ヘルパーを呼ぶと、`views/` 内の `.erb` ファイルを読んで文字列にして返す。Railsのような自動レンダリングは無く、**自分で明示的に呼ぶ**のが基本。

## 役割・なぜ必要か
- ルートの中で「データを用意 → HTMLにして返す」ための層。ロジック（ルート/コントローラ相当）と表示（テンプレート）を分けて、HTMLを散らかさないために使う。
- Sinatraは**テンプレートエンジンを内蔵せず、薄く繋ぐだけ**。標準ライブラリの `erb` がそのまま使える。Haml・Slim・Liquid 等もgemを足せば `haml :index` のように同じ書き味で使える。
- 「明示的に `erb` を呼ぶ」＝ 何が描画されるか透明。Railsの“暗黙のrender”のような魔法が無い。

## 基本の書き方（コード）
```ruby
require "sinatra"

# views/index.erb を描画（拡張子は不要、シンボルで指定）
get "/" do
  @title = "トップ"          # テンプレ内で @title として使える
  erb :index
end

# locals で値を渡す（インスタンス変数を使わず明示的に渡す流儀）
get "/users/:id" do
  user = { id: params[:id], name: "Taro" }
  erb :show, locals: { user: user }   # show.erb 内で user として参照
end

# レイアウトを使わない（部分テンプレ等）
get "/fragment" do
  erb :_row, layout: false
end
```

```erb
<%# views/layout.erb … 全テンプレを囲む“枠”。yield に中身が入る %>
<!DOCTYPE html>
<html><head><title><%= @title %></title></head>
<body><%= yield %></body></html>

<%# views/index.erb … layout の yield に差し込まれる %>
<h1><%= @title %></h1>

<%# views/show.erb … locals で受け取った user を使う %>
<p><%= user[:name] %></p>
```

```ruby
# インラインテンプレート：__END__ 以降に @@名前 で書ける（1ファイル完結向き）
require "sinatra"
get("/") { erb :index }

__END__

@@ layout
<html><body><%= yield %></body></html>

@@ index
<h1>インラインだよ</h1>
```

## 実務での使い方・定番パターン
- **`views/` ディレクトリ**が既定の置き場。`set :views, "templates"` で変更可。
- **`layout.erb`** が存在すれば自動で“枠”として使われる（`yield` に各テンプレが入る）。不要なら `layout: false`。
- **locals 渡し**（`locals: { x: 1 }`）は、どの変数を使うか明示できてテストしやすい。インスタンス変数（`@x`）より推奨されることが多い。
- **インラインテンプレート**（`__END__` ＋ `@@name`）は1ファイルアプリやサンプルで便利。複数ファイルに分けたくなったら `views/` へ移す。
- **部分テンプレ**は `erb :_row, layout: false` のように呼んで組み合わせる（partial専用構文は無く、普通のテンプレを呼ぶだけ）。
- Haml等に乗り換えても `haml :index` と書き方は同じ。エンジンはgem追加で差し替えられる。→ [request_response.md](./request_response.md)

## ハマりどころ / アンチパターン
- **ERBは自動エスケープされない（最重要）**：Railsの `<%= %>` と違い、Sinatra既定のERBは**生HTMLをそのまま出す**。ユーザー入力をそのまま埋めると**XSS**になる。
  - 対策1：`set :erb, escape_html: true`（または `erb :x, escape_html: true`）で `<%= %>` を自動エスケープ化。この設定下では**エスケープしたくない箇所だけ** `<%== %>` で出す。
  - 対策2：個別に `<%= Rack::Utils.escape_html(user_input) %>` で手動エスケープ。
  - 迷ったら `escape_html: true` を**最初から全体に効かせる**のが安全。
- **layoutの取り違え**：`layout.erb` があると全テンプレに自動適用される。API断片やJSON的な出力で枠が混入したら `layout: false` を付け忘れていないか確認。
- **拡張子・置き場のズレ**：`erb :index` は `views/index.erb` を見る。ファイル名/ディレクトリ（`set :views`）がズレると `Errno::ENOENT`。
- **インラインと外部ファイルの混在**で同名テンプレがあるとどちらが効くか分かりにくい。規模が出たら外部ファイルへ寄せる。
- テンプレ内に業務ロジックを詰め込みすぎない。整形はヘルパー（`helpers do ... end`）へ。→ [rack_and_filters.md](./rack_and_filters.md)

## 関連
[request_response.md](./request_response.md) / [rack_and_filters.md](./rack_and_filters.md) / [routing.md](./routing.md)
