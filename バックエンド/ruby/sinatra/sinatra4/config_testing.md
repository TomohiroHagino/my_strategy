# 設定とテスト（config / settings / Rack::Test）

## ひとことで言うと
**設定**＝アプリの振る舞いを環境ごとに切り替える仕組み（`configure` / `set` / `settings`）。
**テスト**＝Sinatraアプリを実際のサーバ起動なしに叩いて検証する仕組み（`Rack::Test`）。
RailsのようにフレームワークがDB・テスト基盤を用意しないので、**両方とも自分で組む**のがSinatra。

## 役割・なぜ必要か
- 環境（development / test / production）で接続先・ログ・例外表示などを変えたい。
- ハードコードを避け、`settings` 経由で一元管理したい。
- リファクタしても壊れていないことを、軽量・高速に確認したい（`Rack::Test` はHTTPサーバを立てずRackアプリへ直接リクエストを流す）。

## 基本の使い方

### 設定（configure / set / settings）
```ruby
require "sinatra"

# 全環境共通の設定
configure do
  set :app_name, "myapp"          # 任意キーを定義
  set :logging, true              # 既存の設定を上書き
  enable :sessions                # set :sessions, true の糖衣
  disable :show_exceptions        # 例外画面を出さない
end

# 特定環境だけの設定（RACK_ENV で切替）
configure :production do
  set :database_url, ENV.fetch("DATABASE_URL")  # 必須envは fetch で fail-fast
  disable :show_exceptions
end

configure :development, :test do
  set :database_url, "sqlite3:db/dev.sqlite3"
end

get "/" do
  settings.app_name               # => "myapp"（settings.foo で参照）
end
```

modular スタイルではクラス内に書く。
```ruby
class App < Sinatra::Base
  configure { set :app_name, "myapp" }
  configure(:production) { disable :show_exceptions }

  get "/name" do
    settings.app_name
  end
end
```

### 環境（RACK_ENV / settings.environment）
```ruby
# 環境の判定
settings.environment           # => :development など（シンボル）
settings.development?          # => true/false（?付きヘルパー）
settings.production?

# 起動時に環境を指定（既定は development）
# $ RACK_ENV=production ruby app.rb
# $ RACK_ENV=test bundle exec rspec
```
- `settings.environment` は環境変数 `RACK_ENV` を読む。未設定なら `:development`。
- `production?` のとき Sinatra は既定で `show_exceptions` を切り、エラーを隠す＝**本番でデバッグ画面が出ないのは正しい挙動**。

### テスト（Rack::Test）
`Rack::Test::Methods` を include すると `get` / `post` などが使え、`last_response` で結果を見る。**`app` メソッドで対象アプリを返す**のが必須。
```ruby
# Gemfile: gem "rack-test", group: :test
require "rack/test"
require_relative "../app"   # 対象アプリ（modular: class App）

ENV["RACK_ENV"] = "test"    # require の前に置くこと（後だと development で読まれる）
```

#### Minitest 例
```ruby
require "minitest/autorun"

class AppTest < Minitest::Test
  include Rack::Test::Methods

  def app                    # ← これが無いと「app未定義」で動かない
    App                      # classic の場合は Sinatra::Application
  end

  def test_root_ok
    get "/"
    assert last_response.ok?            # status 200 か
    assert_equal 200, last_response.status
    assert_includes last_response.body, "myapp"
  end

  def test_post_creates
    post "/items", { name: "x" }
    assert_equal 302, last_response.status   # redirect
    follow_redirect!                          # Location を辿る
    assert last_response.ok?
  end
end
```

#### RSpec 例
```ruby
require "rspec"

RSpec.describe "App" do
  include Rack::Test::Methods
  let(:app) { App }                     # let で app を定義してもよい

  it "returns 200 on root" do
    get "/"
    expect(last_response.status).to eq(200)
    expect(last_response.body).to include("myapp")
  end

  it "passes params" do
    get "/search", q: "ruby"            # 第2引数がクエリ/ボディ
    expect(last_response).to be_ok
  end
end
```

## ハマりどころ / 注意（最重要）
- **`app` を定義し忘れる**：`Rack::Test::Methods` は対象アプリを `app` メソッドから取る。未定義だと `NoMethodError`。classic は `Sinatra::Application`、modular は自分のクラスを返す。
- **`RACK_ENV` の設定タイミング**：アプリ require の**前**に `ENV["RACK_ENV"]="test"` を置く。後だと development で読まれ、`configure :test` が効かない。
- **環境ごとの設定漏れ**：`configure :production` だけに置いた設定は test では未定義。共通は `configure do` に、差分だけ環境別に。
- **`set` の評価タイミング**：`set :x, proc { ... }` のように proc を渡すと参照時に評価。即値なら定義時に固定される。動的な値は proc を使う。
- **`enable :sessions` の暗黙鍵**：セッション秘密鍵を明示しないと再起動で無効化。`set :session_secret, ENV.fetch("SESSION_SECRET")` を本番で必ず指定。

## まとめ（最小実務形）
- 共通は `configure do`、差分は `configure :env do`、参照は `settings.foo`。必須envは `ENV.fetch` で起動時に落とす。
- テストは `rack-test` + `include Rack::Test::Methods` + `app` 定義。`get`/`post` → `last_response.status` / `body` を検証。
- 環境は `RACK_ENV` 駆動。test 環境はアプリ読み込み前に必ず指定。

## 関連
- フィルタ・ミドルウェア・Rackの土台：[rack_and_filters.md](./rack_and_filters.md)
- 始め方（classic / modular）：[getting_started.md](./getting_started.md)
- ハマり所早見表：[pitfalls.md](./pitfalls.md)
