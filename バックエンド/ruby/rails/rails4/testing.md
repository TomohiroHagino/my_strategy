# テスト（Testing）（Rails 4）

## ひとことで言うと
Rails 4 のテストは **Minitest（標準）** か **RSpec（gem）**。`test/` または `spec/` にユニット・機能・統合テストを書く。**system test（ブラウザE2E標準機構）は無い**（Rails 5.1〜）。

## 役割・なぜ必要か
- モデルのバリデーション・ロジック、コントローラの応答、画面遷移を自動で検証し、変更で壊れていないかを担保するためにある。
- リファクタや機能追加の安全網になる。

## 前提（バージョン・構成）
- **標準は Minitest**（Rails 4 で Test::Unit から Minitest ベースに）。`rake test` で実行。
- **RSpec を使うなら `rspec-rails` gem**＋`rails new -T` で標準 test を省く。
- **system test（Capybara統合の標準機構）は Rails 5.1〜**。4 でブラウザテストするなら **Capybara を自前で組み込む**（RSpec feature spec / Minitest 統合テスト）。
- フィクスチャの代わりに **FactoryGirl**（現 FactoryBot。4時代は FactoryGirl 名称）を使うのが定番。

## 基本の書き方（コード）

### Minitest（標準）
```ruby
# test/models/post_test.rb
require "test_helper"

class PostTest < ActiveSupport::TestCase
  test "title is required" do
    post = Post.new(title: nil)
    assert_not post.valid?
  end
end
```
```ruby
# test/controllers/posts_controller_test.rb
class PostsControllerTest < ActionController::TestCase
  test "index renders" do
    get :index
    assert_response :success
  end
end
```
```bash
rake test                 # 全テスト
rake test test/models/post_test.rb
```

### RSpec（gem）
```ruby
# Gemfile（:development, :test）
gem "rspec-rails"
gem "factory_girl_rails"   # 4時代は factory_girl（現 factory_bot）
gem "capybara"             # feature spec（ブラウザ操作）用
```
```ruby
# spec/models/post_spec.rb
require "rails_helper"

RSpec.describe Post, type: :model do
  it "is invalid without a title" do
    post = build(:post, title: nil)   # FactoryGirl
    expect(post).not_to be_valid
  end
end
```
```ruby
# spec/features/post_management_spec.rb（Capybara）
RSpec.feature "Post management" do
  scenario "user creates a post" do
    visit new_post_path
    fill_in "Title", with: "Hello"
    click_button "Create"
    expect(page).to have_content("Hello")
  end
end
```
```ruby
# spec/factories/posts.rb
FactoryGirl.define do
  factory :post do
    title "default title"
    body  "body"
  end
end
```

## 実務での使い方・定番パターン
- **モデルスペック中心**：バリデーション・スコープ・ドメインメソッドを厚く。
- **コントローラスペック / リクエストスペック**：応答コード・リダイレクト・割り当て。RSpec 3 では request spec 推奨の流れ。
- **feature spec（Capybara）でE2E**：system test が無いので Capybara を直接使う。JS が絡む画面は `js: true` ＋ `poltergeist`/`capybara-webkit`/`selenium`。
- **FactoryGirl でテストデータ生成**：fixtures より柔軟。
- **DB クリーニング**：トランザクションロールバック（既定）か `database_cleaner`（JSドライバ併用時）。
- カバレッジは `simplecov`。

## ハマりどころ / アンチパターン
- **system test を探す**：Rails 4 には無い。Capybara を自前導入。
- **`factory_bot` を探す**：4時代は **`factory_girl`** 名称（後に改称）。Gem名に注意。
- **JS テストでDB状態が共有されない**：JS ドライバは別スレッド/プロセスで動くためトランザクションロールバックが効かない。`database_cleaner` を truncation 戦略にする。
- **`assigns` / `render_views`** の挙動差（コントローラスペック）。RSpec のバージョンで非推奨化が進む。
- **テストが実装に密結合**：内部実装でなく振る舞い（入力→出力）を検証する。
- **fixtures とファクトリの混在**で初期データが二重。どちらかに統一。

## 関連
[model.md](./model.md) / [controller.md](./controller.md) / [getting_started.md](./getting_started.md)
