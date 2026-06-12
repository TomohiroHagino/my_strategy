# Service / Form オブジェクト（Rails 4）

## ひとことで言うと
**Service オブジェクト**は複数モデルにまたがる手続き（業務フロー）を1クラスにまとめたもの、**Form オブジェクト**は複数モデルや非DB項目を1つのフォームとして扱うためのオブジェクト。どちらも Fat Model / Fat Controller を防ぐ受け皿。

## 役割・なぜ必要か
- 「注文を確定し在庫を減らしメールを送る」のように**複数モデル＋副作用**が絡む処理を、モデルのコールバックやコントローラに詰めると追えなくなる。これを1つの手続きクラス（Service）に集約する。
- 「ユーザ登録＋プロフィール作成を1フォームで」のように**1フォームが複数モデル/仮想項目**にまたがる時、ActiveModel を使った Form オブジェクトでバリデーションと保存を1箇所にまとめる。
- Rails 4 には Service/Form の専用機構は無い（命名規約だけの素のRubyクラス）。`ActiveModel::Model`（4.0〜）を使うと `validations` や `form_for` 連携が手に入る。

## 基本の書き方（コード）

### Service オブジェクト
```ruby
# app/services/order_placement.rb
class OrderPlacement
  def initialize(user:, cart:)
    @user = user
    @cart = cart
  end

  def call
    ActiveRecord::Base.transaction do
      order = @user.orders.create!(total: @cart.total)
      @cart.items.each { |i| order.lines.create!(item: i) }
      i.product.decrement!(:stock, i.qty) # 在庫を減らす
      OrderMailer.confirmation(order).deliver_later  # 4.2〜。4.0/4.1は別途Sidekiq
      order
    end
  end
end

# コントローラ側は薄く
def create
  @order = OrderPlacement.new(user: current_user, cart: current_cart).call
  redirect_to @order
end
```

### Form オブジェクト（ActiveModel::Model）
```ruby
# app/forms/signup_form.rb
class SignupForm
  include ActiveModel::Model        # 4.0〜。validations と form_for 連携が手に入る

  attr_accessor :email, :password, :company_name

  validates :email, :password, :company_name, presence: true

  def save
    return false unless valid?
    ActiveRecord::Base.transaction do
      company = Company.create!(name: company_name)
      company.users.create!(email: email, password: password)
    end
    true
  end
end
```
```erb
<%= form_for @form, url: signup_path do |f| %>
  <%= f.email_field :email %>
  <%= f.password_field :password %>
  <%= f.text_field :company_name %>
<% end %>
```

## 実務での使い方・定番パターン
- **置き場所**：`app/services/` `app/forms/`。`autoload_paths` に `app/` 配下は自動で入る（Rails 4 はクラシックオートローダ）。
- **命名**：Service は動詞＋名詞（`OrderPlacement` / `UserRegistration`）、メソッドは `call` に統一する流派が多い。
- **トランザクション**で複数モデル更新を包む。失敗時は全ロールバック。
- **メール/重い副作用は非同期へ**：4.2 なら `deliver_later`、4.0/4.1 は Sidekiq を直接。→ [action_mailer.md](./action_mailer.md) / [active_job.md](./active_job.md)
- 戻り値は成功/失敗が分かる形に（オブジェクト or `true/false` or 結果オブジェクト）。

## ハマりどころ / アンチパターン
- **コールバックで代用してしまう**：`after_save` に手続きを詰めると、保存しただけで連鎖が走り追跡不能。複数モデルの手続きはServiceへ。→ [active_record.md](./active_record.md)
- **Service が God クラス化**：1クラスに何でも詰めると肥大。1手続き1クラスに分ける。
- **Form オブジェクトに `ActiveModel::Model` を忘れる**：`form_for` に渡せず、`valid?` も使えない。
- **トランザクション忘れ**：途中失敗で中途半端なデータが残る。複数 `create!` は必ず `transaction` で包む。
- **早すぎる抽象化（YAGNI）**：単純な1モデルCRUDにServiceは過剰。モデルが太ってから導入する。

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
[model.md](./model.md) / [controller.md](./controller.md) / [concern.md](./concern.md) / [active_job.md](./active_job.md)
