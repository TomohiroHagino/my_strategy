# テスト（Testing）（Rails 7）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。Rails標準は **Minitest** だが、実務では **RSpec** が定番。レイヤーごとに model / request / system のテストを書き分ける。

## 役割・なぜ必要か
- 変更のたびに手で全画面を確認するのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- 「この仕様で動く」という実行可能な仕様書になり、リファクタの安全網になる。
- 速くて安定したテスト群があるほど、設計を大胆に変えても怖くない。

## 基本の書き方（コード）
```ruby
# spec/models/user_spec.rb（model spec：バリデーション/ロジック）
RSpec.describe User, type: :model do
  it "emailが空なら無効" do
    user = build(:user, email: nil)   # FactoryBotでテストデータ生成
    expect(user).not_to be_valid
  end
end
```
```ruby
# spec/factories/users.rb（FactoryBot）
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    name { "Taro" }
  end
end
```
```ruby
# spec/requests/posts_spec.rb（request spec：HTTP結合）
RSpec.describe "Posts", type: :request do
  it "一覧が200を返す" do
    create_list(:post, 3)
    get posts_path
    expect(response).to have_http_status(:ok)
  end
end
```
```ruby
# spec/system/login_spec.rb（system spec：Capybaraでブラウザ操作E2E）
RSpec.describe "ログイン", type: :system do
  it "正しい情報でログインできる" do
    user = create(:user, password: "secret")
    visit login_path
    fill_in "Email", with: user.email
    fill_in "Password", with: "secret"
    click_button "Log in"
    expect(page).to have_content("ようこそ")
  end
end
```

## 実務での使い方・定番パターン
- **テストピラミッド**を意識：土台に多数の高速な **model spec**、中間に **request spec**（コントローラ＋ルーティング＋ビューの結合）、頂点に少数の **system spec**（重要フローのE2Eだけ）。
- **FactoryBot** でテストデータを宣言的に生成。`build`（DB保存なし・軽い）と `create`（保存あり）を使い分ける。
- **request spec を主役に**：旧 controller spec より実態に近く、HTTPステータス・リダイレクト・JSONを検証しやすい。
- **system spec** は Capybara + Selenium/Cuprite でブラウザ操作を再現。JS依存画面は `js: true`（ヘッドレスChrome等）。
- DBは各テストでロールバック（`use_transactional_fixtures`）。並列実行は `parallelize` で高速化。

## ハマりどころ / アンチパターン
- **テスト間の状態リーク**：グローバル変数・キャッシュ・外部状態が残り、順序依存で落ちる。`before`/`after` で確実に初期化。
- **過剰なモック**：実装の内部を縛りすぎると、リファクタで赤くなる「もろい」テストに。境界（外部API等）だけモックする。
- **fixtures と factory の混在**：データソースが二重化し管理が破綻。どちらかに統一（実務はfactory寄り）。
- **flakyなsystem spec**：待ち時間依存（`sleep`）で不安定化。Capybaraの**自動待機マッチャ**（`have_content` 等）を使い `sleep` を避ける。
- カバレッジ偏重：数字（80%目安）だけ追って**重要フローのsystem specが無い**のは本末転倒。
- `create` の乱用で遅い → 不要なら `build` / `build_stubbed`。

## 関連
[model.md](./model.md) / [controller.md](./controller.md)
