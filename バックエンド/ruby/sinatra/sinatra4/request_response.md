# リクエスト / レスポンス（Request / Response）（Sinatra）

## ひとことで言うと
ルートブロックの中で使う、**入力（params・request）と出力（status・headers・body・redirect・halt）を扱う道具一式**。「来たものを読む」→「返すものを決める」までをここで完結させる。

## 役割・なぜ必要か
- ルーティングが「どこで処理するか」なら、ここは「**何を読み、何を返すか**」。Webアプリの実処理の入口と出口。
- Sinatraはレスポンスの組み立てが手作業に近く透明。ブロックの**戻り値がそのままボディ**になり、`status`/`headers`/`content_type` で細部を調整する。
- 異常系で処理を即中断したい（認証NG等）ときに `halt`、別URLへ飛ばしたいときに `redirect` を使う。

## 基本の書き方（コード）
```ruby
require "sinatra"
require "json"

# params: クエリ・フォーム・パスパラメータを統合して見られる
post "/users/:id" do
  id   = params[:id]      # パスパラメータ
  name = params[:name]    # フォーム or クエリ
  "id=#{id} name=#{name}"
end

# request オブジェクト（生のHTTP情報）
get "/info" do
  request.request_method   # "GET"
  request.path             # "/info"
  request.user_agent
  request.body.read        # 生ボディ（JSON受信などで使う）
  "ok"
end

# status / headers / body を明示
get "/created" do
  status 201
  headers "X-App" => "demo"
  body "作成しました"
end

# content_type と JSON（内蔵の整形は無いので自前で to_json）
get "/api/user" do
  content_type :json
  { id: 1, name: "Alice" }.to_json
end

# halt: 即座に中断してレスポンスを返す
get "/secret" do
  halt 401, "Unauthorized" unless params[:token] == "ok"
  "secret data"
end

# redirect: 別URLへ
get "/old" do
  redirect "/new", 301
end

# session（要 enable）
enable :sessions          # 上部で1回
get "/login" do
  session[:user] = "alice"
  "logged in"
end
get "/me" do
  session[:user] || "guest"
end
```

## 実務での使い方・定番パターン
- **APIは `content_type :json` ＋ `to_json`** が基本形。共通化するなら `sinatra/json` 拡張や `before` で `content_type` を設定（→ [rack_and_filters.md](./rack_and_filters.md) 相当の filters）。
- **認証ガードは `halt`**：`before` フィルタでトークン検証し、NGなら `halt 401`。ルート内が綺麗に保てる。
- **POST/PUTのJSON受信**は `params` に入らないので `JSON.parse(request.body.read)` で自前パース。
- 成功・失敗で `status` を明示（201/204/400/404/422）。RESTらしいAPIになる。
- `session` はデフォルトでCookieベース。秘密鍵 `set :session_secret, ENV["SESSION_SECRET"]` を必ず設定する。

## ハマりどころ / アンチパターン
- **JSONに内蔵整形は無い**：ハッシュをそのまま返すとRubyの `inspect` 文字列が出る。必ず `.to_json` し、`content_type :json` も付ける（付け忘れでフロントがparse失敗）。
- **`halt` の使い所**：`return` ではブロックを抜けるだけでステータス制御が効かない場面がある。中断＋ステータス指定は `halt` を使う。`before` 内の `halt` はルート実行ごと止める。
- **`params` は文字列キー/シンボルキー両対応**だが、ネストや型変換はしてくれない。数値が欲しければ `params[:id].to_i`。
- **POSTボディは一度しか読めない**：`request.body.read` を2回呼ぶと2回目は空。読んだら変数に保持する。
- **session_secret 未設定**：再起動ごとに鍵が変わりセッションが消える／本番でセキュリティ警告。環境変数で固定する。
- 大きなボディや巨大JSONを `to_json` で一括生成するとメモリを食う。ストリーミングが要るなら `stream do |out| ... end` を検討。

## 関連
[views.md](./views.md) / [routing.md](./routing.md)
