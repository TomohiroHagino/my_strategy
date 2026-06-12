# Service / Form オブジェクト（Rails 7）

## ひとことで言うと
Railsが標準で用意していない、自前で足す2種類のPORO（ただのRubyオブジェクト）。**Service** = 複数モデルにまたがる/外部APIを絡めた「ユースケース1つ」を表す手続きの置き場（`app/services/`）。**Form** = 複数モデルや非永続項目を1つのフォームとしてまとめて扱う受け皿（`ActiveModel::Model` を include）。

## 役割・なぜ必要か
- 「注文を確定する＝在庫を減らし、注文を作り、決済し、メールを送る」のような**複数モデルにまたがる処理**は、特定のモデルにもコントローラにも自然には収まらない。これを置く先がServiceで、**Fat Model / Fat Controller を避けつつ、ユースケース単位でテストできる**ようにする。
- 「1つのフォームで User と Profile を同時に作る」「規約同意チェックなどDBに無い項目も検証したい」ときの受け皿がForm。コントローラやモデルを汚さずに**入力の塊**を扱える。
- どちらも目的は同じ：**責務をはっきりさせて、薄いコントローラ・太りすぎないモデル・テストしやすいコード**を保つこと。

## 基本の書き方（コード）
```ruby
# app/services/orders/checkout.rb （Service：ユースケース1つ＝1オブジェクト）
module Orders
  class Checkout
    def initialize(user:, cart:)
      @user = user
      @cart = cart
    end

    def call
      ActiveRecord::Base.transaction do
        order = @user.orders.create!(total: @cart.total)
        @cart.items.each { |i| order.lines.create!(item: i, qty: i.qty) }
        charge!(order)        # 外部決済API
        OrderMailer.confirmation(order).deliver_later
        order
      end
    end

    private

    def charge!(order) = PaymentGateway.charge(order.total, user: @user)
  end
end
```
```ruby
# 呼び出し側（コントローラは薄く保てる）
order = Orders::Checkout.new(user: current_user, cart: @cart).call
```
```ruby
# app/forms/signup_form.rb （Form：複数モデルを1フォームで）
class SignupForm
  include ActiveModel::Model   # validations / errors / form_with 対応が手に入る

  attr_accessor :email, :password, :nickname, :accept_terms

  validates :email, :password, :nickname, presence: true
  validates :accept_terms, acceptance: true   # DBに無い項目も検証できる

  def save
    return false if invalid?
    ActiveRecord::Base.transaction do
      user = User.create!(email:, password:)
      user.create_profile!(nickname:)
    end
    true
  end
end
```

## 実務での使い方・定番パターン
- **Service名は名詞 or 動詞で「何のユースケースか」を表す**：`Orders::Checkout` / `Users::Invite` のように namespace を切ると整理しやすい。エントリポイントは `call` に統一する流派が多い。
- **トランザクションでまとめる**：複数モデルの更新は `ActiveRecord::Base.transaction` で囲み、途中失敗を巻き戻す。
- **Formは `include ActiveModel::Model`** で `form_with model: @form` がそのまま使え、`@form.errors` もビューで普通に表示できる。
- 戻り値の設計を決めておく（成功/失敗を `true/false` か、結果オブジェクトか）。例外で失敗を表すか戻り値で表すかを揃える。

## ハマりどころ / アンチパターン
- **何でもServiceにして手続き的になる**：まず「それはモデルのメソッドで足りないか」を考える。単一モデルで完結する業務ルールはモデルに置くのが先。→ [model.md](./model.md)
- **God Service（巨大化）**：1つのServiceに分岐や工程を盛りすぎると結局Fat化する。ユースケース単位で小さく分ける。
- **Serviceの中でHTTPの都合（params/flash/redirect）を触る**：それはコントローラの責務。Serviceはドメイン処理に集中させ、入出力はコントローラで。→ [controller.md](./controller.md)
- **共通の属性的振る舞いまでServiceに入れる**：それは Concern 向き。手続き＝Service、クラスに生やす性質＝Concern、と棲み分ける。→ [concern.md](./concern.md)
- **Formにビジネスロジックを溜め込む**：Formは入力の検証と保存の取りまとめが主。重い業務処理はServiceへ委譲する。

## DTO（データ運搬オブジェクト）はどこ？

Rails は **ActiveRecord モデルをそのまま使い回す**のが基本で、Entity と DTO を分けない文化。必要になったら以下で DTO の役割を足す。

| DTOの役割 | Rails での担当 |
|---|---|
| 出力（API/JSON） | Jbuilder（`*.json.jbuilder`）／ ActiveModel::Serializers ／ Alba・blueprinter 等のgem |
| 入力（params検証） | Strong Parameters ＋ 複雑なら **Form オブジェクト**（上記） |
| 画面表示の整形 | Presenter / Decorator（Draper 等） |
| 明示的な値オブジェクト | `Struct` ／ `Data.define`（Ruby 3.2+） |

- 規模が大きくなったら「**AR を直接 view/API に晒さず、Serializer / Form / Presenter を挟む**」形で DTO 層を後付けする。
- Spring のように最初から DTO を分けるのではなく、**ARで兼ねて必要時に足す**のが Rails 流。

## 関連
[model.md](./model.md) / [controller.md](./controller.md) / [concern.md](./concern.md)
