# テスト（Testing）（Rails 5）

## ひとことで言うと
アプリの振る舞いを自動で検証する仕組み。Rails 5 標準は **Minitest**、実務では **RSpec ＋ FactoryBot** が定番。**5.1 からブラウザ操作を伴う system test（Capybara）が標準同梱**された。

## 役割・なぜ必要か
- 手動確認の代わりに「期待する動作」をコードで固定し、変更時のデグレ（壊れ）を素早く検知するためにある。
- モデルのロジック・コントローラの応答・画面操作まで層ごとにテストし、リファクタや機能追加を安全にする。

## テストの種類（層）
- **モデルテスト**：バリデーション・スコープ・ドメインロジック。
- **コントローラ / リクエストテスト**：HTTPの入出力。Rails 5 ではコントローラテストより **request spec / integration test** が推奨方向。
- **system test（5.1〜）**：実ブラウザ（Capybara + Selenium / headless）でユーザ操作を再現。

## 基本の書き方（RSpec 例）
```ruby
# spec/models/user_spec.rb（AAA: Arrange-Act-Assert）
require "rails_helper"

RSpec.describe User, type: :model do
  it "メール未入力なら無効" do
    user = build(:user, email: nil)        # Arrange（FactoryBot）
    expect(user).not_to be_valid           # Act + Assert
  end
end
```
```ruby
# spec/requests/posts_spec.rb（リクエストテスト）
RSpec.describe "Posts", type: :request do
  it "一覧が200で返る" do
    get posts_path
    expect(response).to have_http_status(:ok)
  end
end
```

## system test（Rails 5.1〜・標準同梱）
```ruby
# Minitest 版（test/system/posts_test.rb）
require "application_system_test_case"

class PostsTest < ApplicationSystemTestCase
  test "記事を作成できる" do
    visit new_post_path
    fill_in "Title", with: "テスト"
    click_on "保存"
    assert_text "作成しました"
  end
end
```
- 5.1 で `ActionDispatch::SystemTestCase`（Capybara + Selenium）が標準に入り、JS込みの画面操作テストが書けるようになった。

## 実務での使い方・定番パターン
- **RSpec ＋ FactoryBot ＋ Faker**：テストデータを Factory で用意。`build`（DB保存なし）/ `create`（保存）を使い分ける。
- **テスト名は振る舞いで書く**：「メール未入力なら無効」のように、何を検証しているか分かる名前。
- **request spec を中心に**：Rails 5 ではコントローラ spec より request spec（実際のHTTPに近い）が推奨。
- **system spec / system test で重要フロー**：ログイン→作成→表示などの主要導線をブラウザで通す。
- **DBクリーンアップ**：トランザクションロールバックか `database_cleaner` でテスト間を独立させる。
- **APIモード**は request spec で JSON 応答を検証（`JSON.parse(response.body)`）。→ [api_mode.md](./api_mode.md)

## ハマりどころ / アンチパターン
- **system test は 5.1 から**：5.0 には無い（feature spec を Capybara 手動構成で書く）。
- **テスト間の汚染**：前のテストのデータが残って落ちる。トランザクション/クリーナーで隔離。
- **時刻依存**：`Time.current` を直接使うテストは脆い。`travel_to`（ActiveSupport::Testing::TimeHelpers）で固定。
- **実装に密着しすぎ**：内部メソッドの呼び出し回数を検証すると、リファクタで壊れる。**振る舞い**を検証する。
- **遅い system test を量産**：ブラウザテストは重い。重要フローに絞り、細かい検証はモデル/request specで。

## 関連
[model.md](./model.md) / [controller.md](./controller.md) / [api_mode.md](./api_mode.md)
