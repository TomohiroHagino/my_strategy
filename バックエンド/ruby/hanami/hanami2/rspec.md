# RSpec（Hanami 2）

## ひとことで言うと
Hanami 2 の既定テストフレームワーク。`hanami new` でひな形に組み込まれる。Rails と決定的に違うのは、テスト対象を**コンテナから取得**（`Hanami.app["..."]`）し、依存を **DI（`Deps`）でモックに差し替える**点。グローバルなモデル呼び出しではなく、明示的に注入されたコンポーネントを `describe`/`it` で検証する。

## 役割・なぜ必要か
- Hanami のコンポーネント（アクション・オペレーション・リポジトリ）は依存を `Deps[...]` で受け取るので、**依存を stub すれば単体で隔離テスト**でき、DB や外部 API を叩かずに済む。
- アクション・ビュー・永続化が分離しているため、**レイヤごとに小さく速い** spec を書きやすい。
- 「コンテナから取り出す」「依存を stub する」の2操作と、RSpec の `describe`/`context`/`it`/`let` を覚えれば、Hanami のテストはほぼ網羅できる。

## 基本の書き方（コード）
```ruby
# spec/spec_helper.rb（要点）
ENV["HANAMI_ENV"] ||= "test"
require "hanami/prepare"            # アプリを boot までロード
Hanami.app.container.enable_stubs! # container.stub を使うため有効化（一度だけ）
```
```ruby
# spec/operations/create_user_spec.rb（コンポーネント単体）
RSpec.describe MyApp::Operations::CreateUser do
  # コンテナから取得（依存は自動解決される）
  subject(:operation) { Hanami.app["operations.create_user"] }

  describe "#call" do
    context "正しいメールのとき" do
      it "成功を返す" do
        result = operation.call(email: "a@example.com")
        expect(result).to be_success
      end
    end
  end
end
```
```ruby
# DIで依存を差し替え（コンテナのキーを stub）
RSpec.describe MyApp::Operations::CreateUser do
  let(:user_repo) { instance_double(MyApp::Repositories::UserRepo) }

  before do
    # コンテナのキーをモックに差し替える（enable_stubs! 済み前提）
    Hanami.app.container.stub("repositories.user_repo", user_repo)
  end

  it "リポジトリの create を呼ぶ" do
    allow(user_repo).to receive(:create).and_return(double(id: 1))

    Hanami.app["operations.create_user"].call(email: "a@example.com")

    expect(user_repo).to have_received(:create).with(email: "a@example.com")
  end
end
```
```ruby
# spec/requests/users_spec.rb（リクエスト spec：HTTP結合）
RSpec.describe "Users", type: :request do
  include Rack::Test::Methods         # Rack::Test で app を直接叩く
  let(:app) { Hanami.app }

  it "一覧が200を返す" do
    get "/users"
    expect(last_response.status).to eq(200)
    expect(last_response.body).to include("users")
  end
end
```
```ruby
# アクション単体テスト（コンテナから取得して #call）
RSpec.describe MyApp::Actions::Users::Index do
  subject(:action) { described_class.new }

  it "200を返す" do
    response = action.call({})
    expect(response[0]).to eq(200)   # [status, headers, body]
  end
end
```

## 実務での使い方・定番パターン
- **コンテナからの取得を基本に**：`Hanami.app["operations.create_user"]` のようにキーで取り出す。キーは `app/` のパスに対応（`app/operations/create_user.rb` → `"operations.create_user"`）（→ [dependency_injection.md](./dependency_injection.md)）。
- **`container.stub(key, mock)` で依存差し替え**：`enable_stubs!` を `spec_helper` で一度有効化しておけば、`before` で任意のキーをモックに置換できる。テスト後は自動リセット。
- **`instance_double` で型安全なモック**：`double` ではなく `instance_double(UserRepo)` を使うと、存在しないメソッドの stub を書いたとき即エラーになり、実装ズレを防げる。
- **リクエスト spec は `Rack::Test`**：`Hanami.app` を Rack アプリとして直接叩き、`last_response` でステータス・本文を検証。Capybara を足せばブラウザ E2E も可。
- **DB を使う spec はロールバック**：`database_cleaner-sequel` 等、ROM/Sequel に合うクリーナを使う（ActiveRecord 用ではない）。
- **AAA 構造**：依存を stub（Arrange）→ `call`（Act）→ `expect` で検証（Assert）の素直な3段で書く。

## ハマりどころ / アンチパターン
- **Rails 流の spec をそのまま持ち込む**：`type: :model` でモデルを直接呼ぶ発想は通じない。**コンテナから取得**してテストする設計に切り替える。
- **`stub` の前に `enable_stubs!` を忘れる**：`container.stub` が例外になる。`spec_helper` で一度有効化しておく。
- **コンテナキーの間違い**：パス規約とズレるとコンポーネントが見つからずエラー。`app/operations/create_user.rb` → `"operations.create_user"` を確認する。
- **DI を使わず内部で直接 new**：依存差し替えができず、外部 API や DB を叩く重い・もろいテストになる。依存は必ず注入し、テストで stub する。
- **DB クリーナの取り違え**：ActiveRecord 用設定では効かない。ROM/Sequel 対応のクリーナを選ぶ。
- **過剰なモック**：境界（外部 API・リポジトリ）だけ差し替える。ロジック内部まで縛るとリファクタで赤くなる。

## 関連
[testing.md](./testing.md) / [dependency_injection.md](./dependency_injection.md) / [actions.md](./actions.md)
