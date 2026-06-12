# RSpec（Rails 7）

## ひとことで言うと
Ruby で最も普及している**テストフレームワーク**。`describe`/`context`/`it` で仕様を自然言語に近い形で記述し、`expect(x).to eq(y)` で検証する。Rails 標準は Minitest だが、実務では `rspec-rails` を入れて RSpec で書くのが定番。model / request / system の各 spec をこの上で書く。

## 役割・なぜ必要か
- 「何を・どういう状況で・どう振る舞うか」を `describe`（対象）/`context`（状況）/`it`（期待）の3階層で構造化でき、**テストがそのまま仕様書**になる。
- `let`・`before`・`subject` でテストデータと前提を宣言的に整理でき、重複（DRY）を抑えられる。
- `allow`/`receive` によるモック・スタブを内蔵し、外部 API などの境界を差し替えて隔離テストできる。

## 基本の書き方（コード）
```ruby
# spec/models/user_spec.rb（model spec）
RSpec.describe User, type: :model do
  # subject：検証対象を1つ宣言（明示的に書くと it が読みやすい）
  subject(:user) { build(:user, email: email) }

  describe "#valid?" do
    context "emailがあるとき" do      # 状況をネスト
      let(:email) { "a@example.com" } # 遅延評価：参照時に1度だけ生成
      it { is_expected.to be_valid }  # subject に対するワンライナ
    end

    context "emailが空のとき" do
      let(:email) { nil }
      it "無効になる" do
        expect(user).not_to be_valid
        expect(user.errors[:email]).to be_present
      end
    end
  end
end
```
```ruby
# before：各 example の前に毎回走る前処理
RSpec.describe Cart do
  let(:cart) { Cart.new }
  before { cart.add(item, qty: 2) }   # 各 it の前に実行

  it "合計が計算される" do
    expect(cart.total).to eq(2000)
  end
end
```
```ruby
# マッチャ各種
expect(value).to eq(3)               # == で比較
expect(value).to be > 2              # 比較演算
expect(list).to include("a")         # 含む
expect(list).to match_array([1, 2])  # 順不同で一致
expect { do_it }.to raise_error(ArgumentError)   # 例外
expect { create(:post) }.to change(Post, :count).by(1)  # 副作用
```
```ruby
# allow / receive：モック・スタブ（外部依存の差し替え）
RSpec.describe NotifyUser do
  it "メーラーを呼ぶ" do
    mailer = instance_double(UserMailer)
    allow(UserMailer).to receive(:welcome).and_return(mailer)
    allow(mailer).to receive(:deliver_later)

    NotifyUser.call(user)

    expect(UserMailer).to have_received(:welcome).with(user)
  end
end
```
```ruby
# spec/requests/posts_spec.rb（request spec：HTTP結合）
RSpec.describe "Posts", type: :request do
  it "一覧が200を返す" do
    create_list(:post, 3)            # FactoryBot
    get posts_path
    expect(response).to have_http_status(:ok)
    expect(response.body).to include("記事")
  end
end
```

## 実務での使い方・定番パターン
- **3階層で構造化**：`describe`＝対象（クラス・メソッド）、`context`＝状況（「〜のとき」）、`it`＝期待を1つ。1 example 1 アサーションを基本に。
- **`let` は遅延・`let!` は即時**：`let` は参照時に初めて生成、`let!` は `before` 相当で必ず生成。重い生成は `let` で必要時だけ。
- **`subject` で対象を明示**：検証対象を `subject(:user)` と名付け、`is_expected.to ...` でワンライナにできる。
- **request spec を主役に**：旧 controller spec より実態に近く、ステータス・リダイレクト・JSON を検証しやすい（→ [controller.md](./controller.md)）。
- **system spec で E2E**：重要フローはブラウザ操作で検証。操作は Capybara が担う（→ [capybara.md](./capybara.md)）。
- **`allow` と `expect ... to have_received` の役割分担**：先に `allow` で振る舞いを設定し、後で `have_received` で呼ばれたか検証する。境界（外部 API・メール）だけに使う。

## ハマりどころ / アンチパターン
- **`let` の遅延を忘れる**：`let(:user)` を `before` 内で参照しないと生成されない。前提として必ず作りたいなら `let!` を使う。
- **`context` の説明が状況を表さない**：「正常系」ではなく「ログイン済みのとき」のように**前提条件**を書く。`context` は必ず "when/with" の意味にする。
- **過剰なモック**：`allow ... to receive` で内部メソッドまで縛ると、リファクタで赤くなるもろいテストに。境界だけスタブする。
- **`before` での状態リーク**：グローバル変数・クラス変数・キャッシュが example 間で残ると順序依存で落ちる。各 example で初期化する（DB は `use_transactional_fixtures` で自動ロールバック）。
- **`eq` と `eql`/`equal` の混同**：`eq` は `==`、`eql` は型も一致、`equal` は同一オブジェクト。通常は `eq` を使う。
- **`expect { }` ブロックの付け忘れ**：副作用（`change`）や例外（`raise_error`）は値ではなく**ブロック**を渡す。`expect(do_it).to change(...)` は動かない。

## 関連
[testing.md](./testing.md) / [factory_bot.md](./factory_bot.md) / [capybara.md](./capybara.md)
