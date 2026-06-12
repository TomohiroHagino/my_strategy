# モデル（Model）（Rails 4）

## ひとことで言うと
DBのテーブル1つに対応するクラスで、**データの入れ物＋そのデータに関するビジネスロジックの置き場**。Rails 4 では **`ActiveRecord::Base` を直接継承**する（`ApplicationRecord` は無い＝Rails 5から）。

## 役割・なぜ必要か
- テーブル `users` ↔ クラス `User`、1行 ↔ 1インスタンス、という対応で「SQLを直接書かずにRubyでDBを扱う」ためにある。
- MVCの中で「**何を・どう扱うか（業務ルール）**」を担う中心。コントローラやビューに業務ルールを散らさず、ここへ集約することで再利用・テストがしやすくなる。
- Railsの基本方針は **Fat Model, Skinny Controller**（ロジックはモデル側へ寄せ、コントローラは薄く保つ）。

## 基本の書き方（コード）
```ruby
# app/models/user.rb
class User < ActiveRecord::Base   # 4は ActiveRecord::Base を直接継承
  # 関連
  has_many :posts, dependent: :destroy
  belongs_to :team                 # 4は既定で任意（presenceチェックは付かない）

  # バリデーション
  validates :email, presence: true, uniqueness: { case_sensitive: false }
  validates :name,  length: { maximum: 50 }

  # enum（4.1〜。状態を整数カラムに持たせて名前で扱う）
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
- **Strong Parameters が標準**なので、`attr_accessible`/`attr_protected` はモデルに書かない（書くと `protected_attributes` gem が要る）。許可キーの管理はコントローラ側。→ [strong_parameters.md](./strong_parameters.md)
- **バリデーション**は「アプリ層の入口チェック」。DB側の制約（NOT NULL / unique index）と**二重**に張ると堅い（競合・直接INSERTにも耐える）。
- **関連（association）** で `user.posts` のように辿れる。`dependent:` で親削除時の子の扱いを決める。
- **スコープ**でクエリを意味のある名前に。チェーンできる（`User.admin.recent`）。（`enum` を入れると `User.admin` スコープが生える＝4.1〜）
- **enum** は 4.1 で導入。4.0 には無いので、状態管理は定数＋整数カラムを自前で扱う。
- モデルが太りすぎたら **Concern**（共通の振る舞い）や **Service オブジェクト**（複数モデルにまたがる手続き）へ分割。→ [concern.md](./concern.md) / [service_form.md](./service_form.md)

## ハマりどころ / アンチパターン
- **`belongs_to` が既定で任意**：関連先が nil でも保存できてしまう（7では既定で必須）。必須にしたいなら `validates :team, presence: true` を明示。
- **`enum` は 4.1〜**：4.0 のプロジェクトで `enum` を書くとエラー。バージョンを確認。
- **コールバック詰め込みすぎ**（`after_save` で別モデル更新・メール送信…）→ 隠れた副作用でデバッグ困難。副作用はServiceへ出す。→ [active_record.md](./active_record.md)
- **`save` と `save!` の取り違え**：`save` は失敗時 `false` を返すだけ（握り潰しに注意）、`save!` は例外。
- **モデルにビュー/HTTPの都合を持ち込む**（責務違反）。表示整形はヘルパー/プレゼンターへ。→ [helper.md](./helper.md)
- **N+1** はモデルの使い方（呼び出し側のeager loading）で決まる。→ [active_record.md](./active_record.md)

## 関連
[active_record.md](./active_record.md) / [controller.md](./controller.md) / [helper.md](./helper.md) / [strong_parameters.md](./strong_parameters.md)
