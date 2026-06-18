# DB / Active Record（Rails 5）

## ひとことで言うと
**Active Record** は、モデルとDBをつなぐ **ORM（オブジェクト関係マッピング）**。テーブルをクラス、行をオブジェクトとして扱い、SQLをほぼ書かずにRubyでDBを操作する。

## 役割・なぜ必要か
- DBアクセスを「Rubyのメソッド呼び出し」に翻訳して、生SQLの重複・ミス・SQLインジェクションを避けるためにある。
- スキーマ管理（マイグレーション）・関連（association）・検証（validation）・クエリ・トランザクションまでを一手に担う。

---

## マイグレーションとは
スキーマ変更を**Rubyで記述しバージョン管理**する仕組み。
```ruby
# db/migrate/2017..._create_posts.rb
class CreatePosts < ActiveRecord::Migration[5.2]   # ★ Rails 5 は [5.x] のバージョン指定
  def change
    create_table :posts do |t|
      t.string  :title, null: false
      t.text    :body
      t.references :user, null: false, foreign_key: true
      t.timestamps
    end
    add_index :posts, :title
  end
end
```
- `rails g migration AddXToPosts x:string` → `rails db:migrate` / 戻すなら `rails db:rollback`。
- **Rails 5 のポイント**：`rake db:migrate` でなく **`rails db:migrate`** に統一された。
- **ハマり**: カラム削除など**不可逆/破壊的変更**は段階移行（まず追加→移行→後で削除）。本番は実行順とロック時間に注意。

## アソシエーション（関連）とは
テーブル間の関係を宣言し、`user.posts` のように辿れるようにする。
```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
end
class Post < ApplicationRecord
  belongs_to :user            # ★ Rails 5 では既定で必須（optional: true で任意）
  has_many   :taggings
  has_many   :tags, through: :taggings   # 多対多
end
```
- ポリモーフィック関連（`belongs_to :commentable, polymorphic: true`）、`has_one` もある。
- **Rails 5 注意**：`belongs_to` が必須化されたため、4から移行すると親未設定で保存できなくなる。→ [pitfalls.md](./pitfalls.md)

## バリデーションとは
保存前の値チェック（アプリ層の入口）。
```ruby
validates :email, presence: true, uniqueness: true
validates :age, numericality: { greater_than_or_equal_to: 0 }
```
- **DB制約（NOT NULL / unique index）と二重で**張るのが堅い（unique は競合で破れるため DB側 unique index 必須）。

## コールバックとは
保存・削除等のタイミングで自動実行する処理（`before_save` / `after_create` / `before_destroy` …）。
- **乱用注意**：副作用（メール送信・別モデル更新）を詰めると、保存しただけで何が起きるか追えなくなる。重い副作用は **Service / Active Job** へ。

## スコープとは
よく使うクエリに名前を付ける。
```ruby
scope :published, -> { where(published: true) }
scope :recent,    -> { order(created_at: :desc) }
# 使う: Post.published.recent.limit(10)
```

## Attributes API（Rails 5）
DBカラムに依存しない型付き属性を宣言できる。
```ruby
class Product < ApplicationRecord
  attribute :price, :integer, default: 0     # 型キャスト＋デフォルト
  attribute :tags,  :string, array: true     # 仮想属性にも使える
end
```

## クエリの基本
```ruby
Post.where(published: true).order(created_at: :desc).limit(10)
Post.find(1)            # 無ければ RecordNotFound
Post.find_by(slug: "x") # 無ければ nil
Post.includes(:user)    # eager load（N+1対策）
Post.joins(:comments).where(comments: { approved: true })
Post.pluck(:id)         # 必要な列だけ配列で
Post.find_each { |p| ... }   # 大量データはバッチで
Post.or(Post.where(a: 1), Post.where(b: 2))  # Rails 5 で or が追加
```

## N+1問題とは
一覧表示で関連を1件ずつ引き、SQLが `1 + N` 回走る性能劣化。
```ruby
# NG: posts.each { |p| p.user.name } で N+1
@posts = Post.includes(:user)   # OK: まとめて読む
```
**対策の使い分け**:
- **`includes(:user)`** … 状況に応じ preload/eager_load を自動選択（第一手）。
- **`preload(:user)`** … 常に別クエリ2回（JOINしたくない時）。
- **`eager_load(:user)`** … LEFT JOIN 1クエリ（関連を `where` で絞る一覧）。
- **`joins(:comments)`** … INNER JOIN で絞り込みだけ（関連はロードしない）。
```ruby
Post.eager_load(:user).where(users: { active: true })  # where で関連条件
```
- 検出は **bullet** gem（※ `strict_loading` は 6.1 以降。Rails 5 には無い）。

## トランザクション / ロックとは
```ruby
ActiveRecord::Base.transaction do
  order.save!
  stock.decrement!(:count)        # どれか失敗で全部ロールバック
end
```
- 楽観ロック（`lock_version` カラム）/ 悲観ロック（`with_lock` / `lock!`）。

## ハマりどころ / アンチパターン
- **`belongs_to` 必須化**（Rails 5 最頻）→ 親が任意なら `optional: true`。
- **N+1** → `includes`。
- **`save` と `save!`**：前者は失敗を `false` で返すだけ。握り潰し注意。
- **`default_scope`** の副作用 → 予期せぬ絞り込み。基本使わない（外すには `unscoped`）。
- **time zone**：`Time.now` でなく `Time.current` / `Time.zone.now`。
- マイグレーションとモデルの不整合（カラム消したのにコード参照が残る）。

## ActiveRecord 特有の現象と対策（N+1以外）
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| `count`/`size`/`length` の取り違え | `count`=毎回COUNT、`size`=ロード済みなら配列長、`length`=必ず全件ロード | 一覧後の件数は `size`、件数だけ `count` |
| 関連件数で COUNT 乱発 | `post.comments.count` を一覧ループで | **`counter_cache`** で1カラム参照に |
| 大量データでメモリ枯渇 | `Post.all.each` は全件展開 | **`find_each` / `in_batches`** |
| 不要な全カラム取得 | `Post.all` は全列でモデル生成 | **`pluck` / `select`** |
| 存在確認が重い | `present?` は全件ロード | **`exists?`** |
| コネクションプール枯渇 | Puma スレッド数と `pool` 不一致 | `pool` ≥ スレッド数 |
| `update_all`/`delete_all` で整合崩れ | バリデーション・コールバックをスキップ | 副作用が要るなら1件ずつ |

## 関連
[model.md](./model.md) / [controller.md](./controller.md) / [pitfalls.md](./pitfalls.md)
