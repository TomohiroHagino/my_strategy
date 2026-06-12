# テスト（Rails 6）

## ひとことで言うと
Rails 6 は Minitest を既定とし、並列テスト（`parallelize`）と system test（Capybara + ブラウザ）を標準装備する。実務では RSpec + FactoryBot も定番で、用途に応じて model / request / system のレイヤーを使い分ける。

## 役割・なぜ必要か
- バリデーション・scope・コントローラの応答・主要なユーザーフローを自動検証し、リグレッションを防ぐ。
- Rails 6 で並列テストが標準化され、CPU コア数分のプロセスでテストを分散実行できる。
- system test により、ブラウザ操作（フォーム送信・JS 動作）を含むエンドツーエンドの検証が標準でできる。

## 基本の書き方（コード）

### Minitest（既定）— model / controller / request
```ruby
# test/models/user_test.rb
require "test_helper"

class UserTest < ActiveSupport::TestCase
  test "name is required" do
    user = User.new(name: nil)
    assert_not user.valid?
    assert_includes user.errors[:name], "can't be blank"
  end
end
```
```ruby
# test/integration/users_test.rb（request 相当）
require "test_helper"

class UsersTest < ActionDispatch::IntegrationTest
  test "GET index returns 200" do
    get users_url
    assert_response :success
  end
end
```

### 並列テスト（Minitest）
```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  fixtures :all
end
```
- `:number_of_processors` で CPU コア数分のワーカーを起動する。ワーカーごとに別 DB（`app_test-0`, `app_test-1` ...）が自動作成される。

### system test（標準）
```ruby
# test/application_system_test_case.rb
require "test_helper"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :selenium, using: :headless_chrome, screen_size: [1400, 1400]
end
```
```ruby
# test/system/logins_test.rb
require "application_system_test_case"

class LoginsTest < ApplicationSystemTestCase
  test "user logs in" do
    visit new_session_url
    fill_in "Email", with: "a@example.com"
    fill_in "Password", with: "secret"
    click_on "Log in"
    assert_text "Welcome"
  end
end
```

### RSpec（実務定番）
```ruby
# Gemfile
group :development, :test do
  gem "rspec-rails"
  gem "factory_bot_rails"
end
```
```bash
bundle exec rails generate rspec:install
bundle exec rspec
```
```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name  { "Taro" }
    email { "taro@example.com" }
  end
end
```
```ruby
# spec/requests/users_spec.rb
require "rails_helper"

RSpec.describe "Users", type: :request do
  it "returns 200 on index" do
    create(:user)
    get users_path
    expect(response).to have_http_status(:ok)
  end
end
```
```ruby
# spec/system/login_spec.rb（Capybara）
require "rails_helper"

RSpec.describe "Login", type: :system do
  before { driven_by(:selenium_chrome_headless) }

  it "logs in" do
    user = create(:user)
    visit new_session_path
    fill_in "Email", with: user.email
    click_on "Log in"
    expect(page).to have_text("Welcome")
  end
end
```

## 実務での使い方・定番パターン
- model テスト：バリデーション・scope・カスタムメソッドを検証する。最も数が多くなる層。
- request spec（または IntegrationTest）：エンドポイント単位でステータス・JSON・リダイレクトを検証する。コントローラの結合テストはこちらに寄せる。
- system spec / system test：ログイン・登録・購入など主要フローを最小限に絞ってブラウザで検証する。数を絞り、遅さと不安定さを抑える。
- fixtures（Minitest 標準）はシンプルだが共有状態になりやすい。RSpec 環境では FactoryBot で各テストが必要なデータを明示的に作る方が管理しやすい。
- transactional tests（各テストをトランザクションで囲んでロールバック）が既定。JS を使う system test では別プロセスから DB を見るため、`use_transactional_tests` が効かず DatabaseCleaner（truncation 戦略）を併用することがある。

## ハマりどころ / アンチパターン
- 並列テストでワーカーごとに DB が分かれる：外部サービスやファイルなど DB 外の共有リソースに同時アクセスすると競合する。並列前提で外部接続をモック化するか、競合するスイートは `PARALLEL_WORKERS=1` で直列化する。
- system test が遅い・不安定（flaky）：ブラウザ起動とネットワーク待ちが原因。`assert_text`/`have_text` のような Capybara の自動待機を使い、`sleep` での固定待ちは避ける。本数を主要フローに絞る。
- fixtures の管理：fixtures は全テストで共有されるため、1 箇所の変更が広範囲のテストを壊す。ID 直書きや関連の手書きが破綻しやすい。FactoryBot へ寄せるか、必要最小限に保つ。
- テスト間の状態リーク：クラス変数・グローバル設定・キャッシュ・時刻（`Time.now`）が前のテストの影響を残す。`setup`/`teardown` でリセットし、時刻は `travel_to`（Minitest）/ `Timecop` 等で固定する。
- system test で DB が見えない：JS ドライバ使用時は別スレッド/プロセスで動くため、トランザクション内のデータがブラウザ側から見えないことがある。DatabaseCleaner の truncation 戦略に切り替える。

## 関連
[active_record.md](./active_record.md) / [controller.md](./controller.md) / [routing.md](./routing.md) / [security.md](./security.md) / [javascript.md](./javascript.md) / [pitfalls.md](./pitfalls.md)
