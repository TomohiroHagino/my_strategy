# FactoryBot（Rails 7）

## ひとことで言うと
テスト用のオブジェクト（多くは Active Record モデル）を**宣言的に生成するライブラリ**。`create(:user)` のように書くだけで、定義済みのテンプレートに沿った妥当なデータを生成する。Rails 標準の fixtures より柔軟で、実務の RSpec では事実上の標準。

## 役割・なぜ必要か
- テストごとに `User.new(name: ..., email: ...)` を手で組むのは冗長で、必須項目が増えると破綻する。**factory に既定値を1か所定義**しておけば、各テストは差分だけ指定すればよい。
- DB 保存あり（`create`）／なし（`build`）を選べ、速度と現実性を調整できる。
- `trait`・`association`・`sequence` で「管理者ユーザ」「投稿付きユーザ」「一意な email」などのバリエーションを簡潔に表現できる。

## 基本の書き方（コード）
```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { "Taro" }
    # sequence：呼ぶたびに連番。一意制約のある email に必須
    sequence(:email) { |n| "user#{n}@example.com" }
    role { "member" }

    # trait：名前付きの差分セット
    trait :admin do
      role { "admin" }
    end

    # association：関連を自動生成（投稿を持つユーザ）
    trait :with_posts do
      after(:create) do |user|
        create_list(:post, 3, user: user)
      end
    end
  end

  factory :post do
    title { "タイトル" }
    association :user        # post.user を自動で生成
  end
end
```
```ruby
# 生成メソッドの使い分け
build(:user)          # メモリ上に new するだけ（DB保存なし＝速い）
create(:user)         # DBに保存（id が振られ、関連も保存される）
build_stubbed(:user)  # 保存せず id だけ持つ偽物（最速・DBに触れない）
attributes_for(:user) # 属性のハッシュだけ（パラメータ送信に便利）

# 属性を上書き
create(:user, name: "Jiro", email: "jiro@example.com")

# trait を適用
create(:user, :admin)
create(:user, :admin, :with_posts)

# 複数生成
create_list(:post, 5)
build_list(:user, 3, role: "member")
```
```ruby
# spec での利用（RSpec 設定に include FactoryBot::Syntax::Methods 済み前提）
RSpec.describe Post do
  it "著者が取得できる" do
    user = create(:user, name: "Taro")
    post = create(:post, user: user)
    expect(post.user.name).to eq("Taro")
  end
end
```

## 実務での使い方・定番パターン
- **`rails_helper.rb` で構文を読み込む**：`config.include FactoryBot::Syntax::Methods` を入れると、spec で `create`/`build` を接頭辞なしで呼べる（`FactoryBot.create` の省略）。
- **`build` を優先、必要時だけ `create`**：DB 保存は遅い。バリデーションやメモリ上のロジック検証は `build`、関連やクエリが絡む結合は `create`。読むだけのデータは `build_stubbed` が最速。
- **`sequence` で一意制約を回避**：email・slug など unique 制約のある属性は必ず `sequence` にする。固定値だと2件目でバリデーションに落ちる。
- **`trait` で状態を表現**：`:admin`・`:published`・`:with_posts` のように、状況を trait 名で読めるようにする。`create(:post, :published)` のように組み合わせる。
- **`association` で関連を自動化**：親が必須なモデルは `association :user` を入れておけば、子を作るだけで親も生成される。

## ハマりどころ / アンチパターン
- **`create` の乱用で激遅**：すべて `create` にすると DB I/O が積み重なり CI が分単位で伸びる。保存不要なら `build` / `build_stubbed` に落とす。
- **固定 email でユニーク制約違反**：`email { "a@example.com" }` のように固定すると2件目で落ちる。`sequence` を使う。
- **factory に過剰な既定値**：使わない属性まで盛ると、無関係な変更でテストが壊れる。**バリデーションを通る最小限**にし、必要な属性は各 spec で上書きする。
- **`build` なのに関連を保存前提にする**：`build(:post)` の `post.user` は未保存。保存済み id が要るなら `create` か、関連も含めた `build_stubbed`。
- **fixtures と factory の混在**：データソースが二重化し管理が破綻する。どちらかに統一（実務は factory 寄り）。
- **`after(:create)` の重い処理**：trait で大量の関連を作ると意図せず遅くなる。本当に要るテストだけで使う。

## 関連
[testing.md](./testing.md) / [rspec.md](./rspec.md) / [model.md](./model.md)
