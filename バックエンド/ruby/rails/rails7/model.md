# モデル（Model）（Rails 7）

## ひとことで言うと
DBのテーブル1つに対応するクラスで、**データの入れ物＋そのデータに関するビジネスロジックの置き場**。`ApplicationRecord`（= `ActiveRecord::Base`）を継承する。

## 役割・なぜ必要か
- テーブル `users` ↔ クラス `User`、1行 ↔ 1インスタンス、という対応で「SQLを直接書かずにRubyでDBを扱う」ためにある。
- MVCの中で「**何を・どう扱うか（業務ルール）**」を担う中心。コントローラやビューに業務ルールを散らさず、ここへ集約することで再利用・テストがしやすくなる。
- Railsの基本方針は **Fat Model, Skinny Controller**（ロジックはモデル側へ寄せ、コントローラは薄く保つ）。

## 基本の書き方（コード）
```ruby
# app/models/user.rb
class User < ApplicationRecord
  # 関連
  has_many :posts, dependent: :destroy
  belongs_to :team, optional: true

  # バリデーション
  validates :email, presence: true, uniqueness: { case_sensitive: false }
  validates :name,  length: { maximum: 50 }

  # enum（状態を整数カラムに持たせて名前で扱う）
  enum role: { member: 0, admin: 1 }

  # スコープ（よく使うクエリに名前をつける）
  scope :recent, -> { order(created_at: :desc) }

  # ドメインロジック（業務ルールはここ）
  def display_name
    name.presence || email.split("@").first
  end
end
```

## 実務での使い方・定番パターン
- **バリデーション**は「アプリ層の入口チェック」。DB側の制約（NOT NULL / unique index）と**二重**に張ると堅い（競合・直接INSERTにも耐える）。
- **関連（association）** で `user.posts` のように辿れる。`dependent:` で親削除時の子の扱いを決める。
- **スコープ**でクエリを意味のある名前に。チェーンできる（`User.admin.recent`）。
- **enum** で状態管理（`user.admin?` / `user.admin!`）。
- モデルが太りすぎたら **Concern**（共通の振る舞い）や **Service オブジェクト**（複数モデルにまたがる手続き）へ分割。→ [service_form.md](./service_form.md)

## ハマりどころ / アンチパターン
- **コールバック詰め込みすぎ**（`after_save` で別モデル更新・メール送信…）→ 隠れた副作用でデバッグ困難。副作用はServiceへ出す。→ [active_record.md](./active_record.md)
- **`save` と `save!` の取り違え**：`save` は失敗時 `false` を返すだけ（握り潰しに注意）、`save!` は例外。
- **モデルにビュー/HTTPの都合を持ち込む**（責務違反）。表示整形はヘルパー/プレゼンターへ。→ [helper.md](./helper.md)
- **N+1** はモデルの使い方（呼び出し側のeager loading）で決まる。→ [active_record.md](./active_record.md)
- `belongs_to` は既定で必須（presence）。任意にするなら `optional: true`。

## 関連
[active_record.md](./active_record.md) / [controller.md](./controller.md) / [helper.md](./helper.md)
