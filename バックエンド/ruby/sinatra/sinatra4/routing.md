# ルーティング（Routing）（Sinatra）

## ひとことで言うと
**「HTTPメソッド ＋ URLパターン」と「処理ブロック」を結びつける仕組み**。`get "/" do ... end` のように書く。Sinatraの中核で、リクエストが来たら**上から順に最初にマッチしたルート**が実行される。

## 役割・なぜ必要か
- Webアプリの仕事は「どのURLにどのメソッドで来たら、何を返すか」を決めること。ルーティングはその対応表そのもの。
- Sinatraはルータが薄く透明なので、**定義した順序がそのまま優先順位**になる。Railsのような複雑なルート解決の「魔法」が無い分、挙動が読みやすい。
- パスパラメータ・ワイルドカード・正規表現で、固定URLだけでなく動的なURLも捕まえられる。

## 基本の書き方（コード）
```ruby
require "sinatra"

# HTTPメソッドごとにブロックを定義
get    "/users"       do "一覧"      end
post   "/users"       do "作成"      end
put    "/users/:id"   do "全更新"    end
patch  "/users/:id"   do "部分更新"  end
delete "/users/:id"   do "削除"      end

# パスパラメータ: :id → params[:id] で取れる
get "/users/:id" do
  "user = #{params[:id]}"
end

# ワイルドカード * → params['splat'] に配列で入る
get "/files/*.*" do
  # /files/foo/bar.png → splat == ["foo/bar", "png"]
  name, ext = params["splat"]
  "name=#{name} ext=#{ext}"
end

# 正規表現ルート（マッチ結果も splat に入る）
get %r{/posts/(\d+)} do
  id = params["captures"].first   # または params["splat"]
  "post #{id}"
end

# クエリ文字列も同じ params から
get "/search" do
  # /search?q=ruby&page=2
  "q=#{params[:q]} page=#{params[:page]}"
end
```

## 実務での使い方・定番パターン
- **REST的に並べる**：`/users`(get/post)、`/users/:id`(get/patch/delete) を素直に列挙。Sinatraには `resources` のような一括定義は無いので手で書く。
- **具体的なルートを先、曖昧なルートを後**に置く。例：`/users/new` は `/users/:id` より**上**に書かないと `:id == "new"` に食われる。
- 任意パラメータは `?` 付き：`get "/posts/:id?"` で `/posts` も `/posts/5` も拾える。
- **条件（conditions）** で絞り込み：`get "/", host_name: /^admin\./ do ... end` のようにホストやUAで分岐できる。
- マッチしたら `params` から値を取り、処理して文字列（=レスポンスボディ）を返すのが基本形。詳細は → [request_response.md](./request_response.md)

## ハマりどころ / アンチパターン
- **定義順の事故**：`/users/:id` を先に書くと `/users/new` や `/users/search` まで `:id` で捕まる。**固定パスを先・動的パスを後**が鉄則。
- **末尾スラッシュ問題**：Sinatraは `/users` と `/users/` を**別物**として扱う。両方受けたいなら `get "/users/?" do`（末尾`/`を任意に）と書く。
- **ワイルドカードは `params[:splat]` ではなく `params['splat']`**（配列）。`:id` のような名前付きと取り出し方が違う点に注意。
- マッチしたルートが見つからないと **404（Sinatra::NotFound）**。`not_found do ... end` で共通ハンドラを用意する。
- ルートをだらだら1ファイルに書き続けると保守不能に。増えたら modular 化や機能別ファイル分割を検討（→ [getting_started.md](./getting_started.md)）。

## 関連
[request_response.md](./request_response.md) / [getting_started.md](./getting_started.md)
