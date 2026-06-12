# モデル（Rails 6）

## ひとことで言うと
`app/models` に置き `ApplicationRecord` を継承するActive Recordのクラス。1モデル＝1テーブルが基本で、バリデーション・関連・スコープ・コールバック・enum を定義してデータの整合性と取得ロジックをまとめる。

## 役割・なぜ必要か
データの保存ルール（バリデーション）はDBではなくモデルに集約することで、保存前に検証してエラーメッセージを返せる。テーブル間の関係（関連）を宣言すると `post.comments` のように辿れる。クエリの定型はscopeで名前付き再利用できる。

## 基本の書き方（コード）
```ruby
class Post < ApplicationRecord
  # 関連
  belongs_to :user                                   # posts.user_id（既定で存在必須）
  has_many   :comments, dependent: :destroy          # 親削除時に子も削除
  has_one    :summary
  has_many   :taggings
  has_many   :tags, through: :taggings               # 中間テーブル経由の多対多

  # enum（DBは integer 列。status: 0=draft, 1=published…）
  enum status: { draft: 0, published: 1, archived: 2 }

  # バリデーション
  validates :title, presence: true, length: { maximum: 100 }
  validates :body,  presence: true
  validates :slug,  uniqueness: true

  # スコープ（クエリの再利用）
  scope :recent,    -> { order(created_at: :desc) }
  scope :published, -> { where(status: :published) }
  scope :by_user,   ->(user) { where(user: user) }

  # コールバック（保存前に正規化など）
  before_validation :normalize_title

  private

  def normalize_title
    self.title = title.to_s.strip
  end
end
```
```ruby
post = Post.new(title: "  Hello ", body: "本文", user: current_user)
post.valid?        # => true（before_validation で trim 済み）
post.save          # 検証通過で INSERT
post.published!    # enum のbangメソッドで status を published に更新
Post.published.recent.by_user(current_user)   # scope はチェーン可能
```

## 実務での使い方・定番パターン
- バリデーションは「必ずモデル」に置く。コントローラやビューで検証しない（複数の入口から保存されても効くため）。
- 繰り返す `where`/`order` はscope化して名前で呼ぶ。引数つきscopeはlambdaで書く。
- enumはステータス系に有効。`post.published?` 述語と `Post.published` scopeが自動で生える。
- 一覧で `post.comments` を回すなら `includes(:comments)` でN+1を防ぐ（詳細は [active_record.md](./active_record.md)）。

## ハマりどころ / アンチパターン
- コールバック乱用：保存時に外部API送信やメール配信を `after_save` に詰めると、テストや一括更新で副作用が暴発する。副作用はサービスやジョブへ寄せる（[service_form.md](./service_form.md) / [active_job.md](./active_job.md)）。
- バリデーションをスキップするsave系に注意：`update_column` `update_columns` `save(validate: false)` `update_all` `insert_all`/`upsert_all` は検証もコールバックも走らない。不正データ混入の原因になる。
- `belongs_to` はRails 5以降、既定で存在必須。任意にするなら `optional: true`。
- Fat Model：1モデルに数百行のロジックが集まったら、concernやサービスへ分割する（[concern.md](./concern.md)）。複雑なクエリ詳細は [active_record.md](./active_record.md) を参照。

## 関連
[active_record.md](./active_record.md) / [controller.md](./controller.md) / [concern.md](./concern.md) / [service_form.md](./service_form.md) / [active_job.md](./active_job.md) / [console.md](./console.md)
