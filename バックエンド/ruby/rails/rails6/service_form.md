# Service / Form オブジェクト（Rails 6）

## ひとことで言うと
Fat Controller・Fat Model を避けるため、業務ロジックを PORO（素の Ruby クラス）に切り出す。複数モデルをまたぐ処理は Service、複数モデルをまたぐフォームは Form Object にする。

## 役割・なぜ必要か
- コントローラに業務ロジックを書くと肥大化し、テストもしづらい。モデルに何でも書くと 1 モデルが巨大化する。
- Service：複数モデルの更新・外部API呼び出し・トランザクションをまとめる処理単位。`app/services` に置く。
- Form Object：1 フォームで複数モデルを同時に作成・更新するときの受け皿。`ActiveModel::Model` を include してバリデーションを持つ。

## 基本の書き方（コード）
```ruby
# app/services/order_creator.rb
class OrderCreator
  def initialize(user:, cart:)
    @user = user
    @cart = cart
  end

  def call
    ActiveRecord::Base.transaction do
      order = @user.orders.create!(total: @cart.total)
      @cart.items.each { |i| order.line_items.create!(product: i.product, qty: i.qty) }
      @cart.clear!
      order
    end
  rescue ActiveRecord::RecordInvalid => e
    Rails.logger.error(e.message)
    nil
  end
end

# 呼び出し（コントローラ）
order = OrderCreator.new(user: current_user, cart: @cart).call
```
```ruby
# app/forms/signup_form.rb
class SignupForm
  include ActiveModel::Model

  attr_accessor :name, :email, :password, :company_name

  validates :name, :email, :company_name, presence: true

  def save
    return false if invalid?
    ActiveRecord::Base.transaction do
      company = Company.create!(name: company_name)
      User.create!(name: name, email: email, password: password, company: company)
    end
    true
  rescue ActiveRecord::RecordInvalid
    false
  end
end

# コントローラ
@form = SignupForm.new(signup_params)
if @form.save
  redirect_to root_path
else
  render :new
end
```

## 実務での使い方・定番パターン
- 決済・登録・退会のような複数ステップ＋トランザクションが必要なフローは Service。
- 「ユーザー＋会社」を 1 画面で同時登録するような複数モデルフォームは Form Object（`form_with model: @form` で error 表示も使える）。
- Service の戻り値は成否が分かる形（オブジェクト or nil、または Result オブジェクト）に統一する。

## ハマりどころ / アンチパターン
- 1 メソッドを呼ぶだけの薄いラッパー Service を量産すると無意味。複数モデル操作やトランザクションが絡むときだけ作る。
- Form Object のバリデーションは `ActiveModel::Model` の `validates` を使う。DB制約だけに頼らない。
- 例外で失敗を伝えるか戻り値で伝えるかを混在させると呼び出し側が混乱する。プロジェクトで方針を統一する。

## DTO（データ運搬オブジェクト）はどこ？

Rails は **ActiveRecord モデルをそのまま使い回す**のが基本で、Entity と DTO を分けない文化。必要になったら以下で DTO の役割を足す。

| DTOの役割 | Rails での担当 |
|---|---|
| 出力（API/JSON） | Jbuilder（`*.json.jbuilder`）／ ActiveModel::Serializers（AMS） |
| 入力（params検証） | Strong Parameters ＋ 複雑なら **Form オブジェクト**（上記） |
| 画面表示の整形 | Presenter / Decorator（Draper 等） |
| 明示的な値オブジェクト | `Struct`（値オブジェクト） |

- 規模が大きくなったら「**AR を直接 view/API に晒さず、Serializer / Form / Presenter を挟む**」形で DTO 層を後付けする。
- Spring のように最初から DTO を分けるのではなく、**ARで兼ねて必要時に足す**のが Rails 流。

## 関連
[controller.md](./controller.md) / [model.md](./model.md) / [concern.md](./concern.md) / [active_record.md](./active_record.md) / [strong_parameters.md](./strong_parameters.md) / [testing.md](./testing.md) / [pitfalls.md](./pitfalls.md)
