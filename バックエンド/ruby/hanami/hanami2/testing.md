# テスト（Testing）（Hanami 2）

## ひとことで言うと
Hanami 2 のテストは **RSpec が事実上の標準**。Rails と決定的に違うのは、**テスト対象をコンテナから取得**（`Hanami.app["..."]`）し、**DI で依存をモックに差し替える**点。グローバルな `Post.find` ではなく、明示的に注入されたコンポーネントを検証する。

## 役割・なぜ必要か
- 各コンポーネントは依存を `Deps[...]` で受け取るので、**依存を差し替えれば単体で隔離テスト**できる（DBや外部APIを叩かずに済む）。
- アクション・ビュー・永続化が分離しているため、**レイヤごとに小さく速いテスト**を書きやすい。
- 「コンテナから取り出す」「依存を stub する」という2操作を覚えれば、Hanami のテストはほぼ網羅できる。

## 基本の書き方（コード）
```ruby
# spec/operations/create_user_spec.rb（コンポーネント単体）
RSpec.describe MyApp::Operations::CreateUser do
  # コンテナから取得（依存は自動解決される）
  subject(:operation) { Hanami.app["operations.create_user"] }

  it "ユーザを作成する" do
    result = operation.call(email: "a@example.com")
    expect(result).to be_success
  end
end
```
```ruby
# DIで依存を差し替え（モック登録）
RSpec.describe MyApp::Operations::CreateUser do
  let(:user_repo) { instance_double(MyApp::Repositories::UserRepo) }

  before do
    # コンテナのキーをモックに差し替える（stub）
    Hanami.app.container.stub("repositories.user_repo", user_repo)
  end

  it "リポジトリの create を呼ぶ" do
    allow(user_repo).to receive(:create).and_return(double(id: 1))
    Hanami.app["operations.create_user"].call(email: "a@example.com")
    expect(user_repo).to have_received(:create)
  end
end
```
```ruby
# spec/requests/users_spec.rb（リクエストテスト：HTTP結合）
RSpec.describe "Users", type: :request do
  include Rack::Test::Methods            # Rack::Test で app を叩く
  let(:app) { Hanami.app }

  it "一覧が200を返す" do
    get "/users"
    expect(last_response.status).to eq(200)
  end
end
```
```ruby
# spec/spec_helper.rb（要点）
ENV["HANAMI_ENV"] ||= "test"
require "hanami/prepare"               # アプリをロード（boot まで）
# container.stub を使うため dry-system の stubs を有効化
Hanami.app.container.enable_stubs!
```

## 実務での使い方・定番パターン
- **コンテナからの取得を基本に**：`Hanami.app["operations.create_user"]` のようにキー（パス規約）で取り出す。キーは `app/` のパスに対応（→ [dependency_injection.md](./dependency_injection.md)）。
- **依存は `container.stub(key, mock)` で差し替え**：`enable_stubs!` を一度有効化しておけば、`before` ブロックで任意のキーをモックに置換できる。テスト後は自動リセット。
- **リクエストテストは `Rack::Test`**：`Hanami.app` を Rack アプリとして直接叩き、ステータス・本文・リダイレクトを検証。Capybara を足せばブラウザE2Eも可。
- **DB を使うテストはトランザクションでロールバック**：`database_cleaner-sequel` 等、ROM/Sequel に合うクリーナを使う（ActiveRecord 用ではない）。
- **AAA（Arrange-Act-Assert）**：依存をモック登録（Arrange）→ `call`（Act）→ 期待を検証（Assert）の素直な3段で書く。

## ハマりどころ / アンチパターン
- **Rails 流のテストをそのまま持ち込む**：`type: :model` で `User` を直接呼ぶ発想は通じない。**コンテナから取得**してテストする設計に切り替える。
- **`stub` を呼ぶ前に `enable_stubs!` を忘れる**：`container.stub` が例外になる。`spec_helper` で一度有効化しておく。
- **`Hanami.app["..."]` のキー間違い**：パス規約とズレるとコンポーネントが見つからずエラー。`app/operations/create_user.rb` → `"operations.create_user"`。
- **DIを使わず内部を直接 new**：依存差し替えができず、外部APIやDBを叩く重い・もろいテストに。依存は必ず注入し、テストで stub する。
- **DBクリーナの取り違え**：ActiveRecord 用設定では効かない。ROM/Sequel 対応のクリーナを選ぶ。
- **過剰なモック**：境界（外部API・リポジトリ）だけ差し替える。ロジック内部まで縛るとリファクタで赤くなる。

## 関連
[rspec.md](./rspec.md) / [dependency_injection.md](./dependency_injection.md) / [settings_config.md](./settings_config.md) / [actions.md](./actions.md)
