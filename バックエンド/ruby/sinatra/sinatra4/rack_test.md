# Rack::Test（Sinatra 4）

## ひとことで言うと
HTTP サーバを**起動せずに** Rack アプリへ直接リクエストを流すテスト用ライブラリ。`get "/"` のように書くと内部でアプリを呼び、結果を `last_response` で受け取れる。Sinatra はテスト基盤を内蔵しないため、アプリを直接叩く検証はこの `rack-test` gem で組むのが定番。

## 役割・なぜ必要か
- Sinatra アプリは Rack アプリそのもの。`Rack::Test` は実サーバ（Puma 等）を立てずにアプリへリクエストを渡せるので、**軽量・高速**にルーティングやレスポンスを検証できる。
- ステータス・本文・ヘッダ・リダイレクトを `last_response` で確認でき、API や Webhook 受け口のテストに十分。
- Minitest でも RSpec でも `include Rack::Test::Methods` するだけで `get`/`post` ヘルパが使える。フレームワーク非依存で使い回せる。

## 基本の書き方（コード）
```ruby
# Gemfile
# gem "rack-test", group: :test

ENV["RACK_ENV"] = "test"      # アプリ require の「前」に置く（後だと development で読まれる）
require "rack/test"
require_relative "../app"     # 対象アプリ（modular なら class App）
```
```ruby
# RSpec 例
require "rspec"

RSpec.describe "App" do
  include Rack::Test::Methods

  # app メソッドが必須：対象の Rack アプリを返す
  def app
    App                       # classic スタイルなら Sinatra::Application
  end

  it "ルートが200を返す" do
    get "/"
    expect(last_response).to be_ok            # status 200 か
    expect(last_response.status).to eq(200)
    expect(last_response.body).to include("myapp")
  end

  it "クエリを渡せる" do
    get "/search", q: "ruby"                  # 第2引数がクエリ/ボディ
    expect(last_response).to be_ok
  end

  it "POSTでリダイレクトする" do
    post "/items", { name: "x" }
    expect(last_response.status).to eq(302)
    follow_redirect!                          # Location を辿る
    expect(last_response).to be_ok
  end
end
```
```ruby
# Minitest 例
require "minitest/autorun"

class AppTest < Minitest::Test
  include Rack::Test::Methods

  def app                     # ← 無いと NoMethodError で動かない
    App
  end

  def test_root_ok
    get "/"
    assert last_response.ok?
    assert_includes last_response.body, "myapp"
  end
end
```
```ruby
# ヘッダ・JSON・セッションの検証
header "Authorization", "Bearer xxx"    # リクエストヘッダを付ける
post "/api/items", { name: "x" }.to_json, "CONTENT_TYPE" => "application/json"
expect(last_response.headers["Content-Type"]).to include("application/json")
json = JSON.parse(last_response.body)
expect(json["name"]).to eq("x")
```

## 実務での使い方・定番パターン
- **`app` メソッドが起点**：`Rack::Test::Methods` は対象アプリを `app` から取る。modular は自分のクラス（`App`）、classic は `Sinatra::Application` を返す。RSpec では `def app` でも `let(:app)` でもよい。
- **`RACK_ENV=test` はアプリ読込前**：`require` より前に `ENV["RACK_ENV"]="test"` を置く。後だと development 設定で読まれ、`configure :test` が効かない（→ [config_testing.md](./config_testing.md)）。
- **`get`/`post` の第2引数はパラメータ**：クエリやフォームボディとして渡る。JSON を送るなら文字列ボディ＋`CONTENT_TYPE` ヘッダ。
- **`follow_redirect!` で遷移を追う**：302 を返すエンドポイントは `follow_redirect!` で Location を辿り、最終ページを検証する。
- **`last_request`/`last_response`**：直近のリクエスト・レスポンスを参照する。`last_response.status`・`.body`・`.headers` が検証の中心。
- **DB が要るなら自前で用意**：Sinatra は ORM もテスト DB も内蔵しない。Sequel/ActiveRecord を組み、test 環境用の接続とクリーニングを自分で設定する（→ [data_access.md](./data_access.md)）。

## ハマりどころ / アンチパターン
- **`app` の定義忘れ（最頻）**：未定義だと `NoMethodError`。modular は自分のクラス、classic は `Sinatra::Application` を返す。
- **`RACK_ENV` の設定タイミング**：アプリ require の後に設定すると development で読まれ、test 用設定が無視される。必ず require より前。
- **classic と modular の取り違え**：1ファイル classic は `Sinatra::Application`、`Sinatra::Base` を継承した modular は自作クラスを `app` に返す。混同すると空のアプリを叩く。
- **セッション秘密鍵未設定**：`enable :sessions` でセッションを使うテストは、秘密鍵が無いとリクエスト間で状態が保てない。test でも `set :session_secret, "..."` を指定する。
- **JS/ブラウザ挙動は検証できない**：`Rack::Test` は HTTP レベルのみ。JS 実行やクリック操作が要るなら Capybara を足す（Sinatra でも利用可）。
- **`get "/search", q: "x"` のハッシュをヘッダと混同**：第2引数はパラメータ、第3引数が Rack 環境（ヘッダ）。順番を間違えるとパラメータがヘッダ扱いになる。

## 関連
[config_testing.md](./config_testing.md) / [rack_and_filters.md](./rack_and_filters.md) / [getting_started.md](./getting_started.md)
