# Service / Form オブジェクト（Rails 5）

## ひとことで言うと
**Service オブジェクト**は「複数モデルにまたがる手続き」を1クラスに切り出したもの、**Form オブジェクト**は「複数モデルや複数入力を1つのフォームとして扱う」ためのクラス。どちらも Fat Model / Fat Controller を防ぐ設計パターン。

## 役割・なぜ必要か
- 「ユーザ登録して→ウェルカムメール送って→初期データ作る」のような**手続き（ユースケース）**は、特定の1モデルに属さない。コントローラに書くと太り、モデルのコールバックに詰めると副作用が隠れる。これを Service に出す。
- 1つのフォームで複数モデル（User と Profile）や DBに無い項目（規約同意チェック）を扱いたいとき、Form オブジェクトで「画面用の1モデル」を作る。

## Service オブジェクトの基本
```ruby
# app/services/user_registration.rb
class UserRegistration
  def initialize(params)
    @params = params
  end

  def call
    user = nil
    ActiveRecord::Base.transaction do          # まとめて成功/失敗
      user = User.create!(@params)
      user.create_profile!
    end
    WelcomeMailer.welcome(user).deliver_later  # コミット後に送る方が安全
    user
  end
end

# コントローラから
def create
  @user = UserRegistration.new(user_params).call
  redirect_to @user
rescue ActiveRecord::RecordInvalid
  render :new
end
```

## Form オブジェクトの基本
```ruby
# app/forms/signup_form.rb
class SignupForm
  include ActiveModel::Model        # validations / バリデーションが使えるようになる

  attr_accessor :name, :email, :password, :accept_terms

  validates :name, :email, presence: true
  validates :accept_terms, acceptance: true     # DBに無い項目も検証できる

  def save
    return false unless valid?
    User.create!(name: name, email: email, password: password)
  end
end
```
- `ActiveModel::Model` を include すると、ARモデルでないクラスでも `form_with`/`form_for` やバリデーションが使える。

## 実務での使い方・定番パターン
- **Service は `call` 1メソッド**に統一すると呼び出し側が読みやすい（命名規約として広く使われる）。
- **トランザクション境界を Service に持たせる**：複数の `save!` をまとめ、失敗で全ロールバック。
- **ジョブ/メール送信はコミット後**：トランザクション内で `perform_later` すると「まだ無いレコード」を引いて失敗する。→ [active_job.md](./active_job.md)
- **Form オブジェクトで `accepts_nested_attributes_for` の複雑さを回避**：ネスト属性が辛くなったら Form に寄せる。
- 戻り値で成否を返す（`true/false` か例外）かを統一して、コントローラの分岐を単純化。

## ハマりどころ / アンチパターン
- **Service の乱造**：何でもかんでも Service にすると、1モデルで足りる処理まで散らかる。複数モデルにまたがる/手続き的なものに限る。
- **Service にHTTP/ビューの都合を持ち込む**：`flash` や `redirect` を Service でやらない。応答はコントローラの責務。
- **Form オブジェクトに永続化を詰めすぎ**：保存ロジックが膨らんだら Service と併用する。
- **`call` の中で例外を握り潰す**：失敗が呼び出し側に伝わらない。例外か戻り値で必ず成否を返す。

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
[model.md](./model.md) / [concern.md](./concern.md) / [active_job.md](./active_job.md)
