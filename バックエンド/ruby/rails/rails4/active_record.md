# DB / Active Record（Rails 4）

## ひとことで言うと
**Active Record** は、モデルとDBをつなぐ **ORM（オブジェクト関係マッピング）**。テーブルをクラス、行をオブジェクトとして扱い、SQLをほぼ書かずにRubyでDBを操作する。

## 役割・なぜ必要か
- DBアクセスを「Rubyのメソッド呼び出し」に翻訳して、生SQLの重複・ミス・SQLインジェクションを避けるためにある。
- スキーマ管理（マイグレーション）・関連（association）・検証（validation）・クエリ・トランザクションまでを一手に担う。

---

## マイグレーションとは
スキーマ変更を**Rubyで記述しバージョン管理**する仕組み。
```ruby
# db/migrate/2015..._create_posts.rb
class CreatePosts < ActiveRecord::Migration   # 4は世代番号 [x.y] を付けない（5.0から付く）
  def change
    create_table :posts do |t|
      t.string  :title, null: false
      t.text    :body
      t.references :user, index: true, foreign_key: true   # foreign_key は4.2〜
      t.timestamps null: false                             # 4.2は明示推奨（DEPRECATION回避）
    end
    add_index :posts, :title
  end
end
```
- `rails g migration AddXToPosts x:string` → `rake db:migrate` / 戻すなら `rake db:rollback`。
- **`foreign_key: true` は 4.2 から**。4.0/4.1 では外部キー制約は付けられない（gem `foreigner` 等で補う）。
- **`t.timestamps null: false`**：4.2 では `null:` を明示しないと将来非推奨の警告が出る。
- **ハマり**: カラム削除など**不可逆/破壊的変更**は段階移行（まず追加→移行→後で削除）。本番は実行順とロック時間に注意。

## アソシエーション（関連）とは
テーブル間の関係を宣言し、`user.posts` のように辿れるようにする。
```ruby
class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy
end
class Post < ActiveRecord::Base
  belongs_to :user            # 4は既定で任意（presenceチェックは付かない）
  has_many   :taggings
  has_many   :tags, through: :taggings   # 多対多
end
```
- ポリモーフィック関連（`belongs_to :commentable, polymorphic: true`）、`has_one` もある。

## バリデーションとは
保存前の値チェック（アプリ層の入口）。
```ruby
validates :email, presence: true, uniqueness: true
validates :age, numericality: { greater_than_or_equal_to: 0 }
```
- **DB制約（NOT NULL / unique index）と二重で**張るのが堅い（uniqueは競合で破れるためDB側 unique index 必須）。

## コールバックとは
保存・削除等のタイミングで自動実行する処理（`before_save` / `after_create` / `before_destroy` …）。
- **乱用注意**：副作用（メール送信・別モデル更新）を詰めると、保存しただけで何が起きるか追えなくなる。重い副作用は **Service** や（4.2以降は）**Active Job** へ。4.0/4.1 では Sidekiq 等を直接呼ぶ。→ [active_job.md](./active_job.md)

## スコープとは
よく使うクエリに名前を付ける。
```ruby
scope :published, -> { where(published: true) }
scope :recent,    -> { order(created_at: :desc) }
# 使う: Post.published.recent.limit(10)
```
- Rails 4 では `scope` の第2引数は **必ず lambda（`->`）**。Rails 3 の `scope :x, where(...)` 形式は使えない。

## クエリの基本
```ruby
Post.where(published: true).order(created_at: :desc).limit(10)
Post.find(1)            # 無ければ RecordNotFound
Post.find_by(slug: "x") # 無ければ nil（4で find_by が正式メソッド）
Post.includes(:user)    # eager load（N+1対策）
Post.joins(:comments).where(comments: { approved: true })
Post.pluck(:id)         # 必要な列だけ配列で
Post.find_each { |p| ... }   # 大量データはバッチで
Post.not_published = Post.where.not(published: true)  # where.not は4〜
```
- **`where.not`**、**`find_or_create_by`**、**`references`** は Rails 4 で導入/整理された。
- 動的ファインダ（`find_all_by_title`）は Rails 4 で非推奨。`where` / `find_by` に置き換える。

## N+1問題とは
一覧表示で関連を1件ずつ引き、SQLが `1 + N` 回走る性能劣化。
```ruby
# NG: posts.each { |p| p.user.name } で N+1
@posts = Post.includes(:user)   # OK: まとめて読む
```
**対策の使い分け**:
- **`includes`** … preload/eager_load を自動選択（第一手）。
- **`preload`** … 常に別クエリ2回。**`eager_load`** … LEFT JOIN 1クエリ（`where` で関連を絞る時）。
- **`joins`** … INNER JOIN で絞り込みだけ（関連はロードしない）。
- 検出は **bullet** gem（`strict_loading` は 6.1 以降で Rails 4 には無い）。

## トランザクション / ロックとは
```ruby
ActiveRecord::Base.transaction do
  order.save!
  stock.decrement!(:count)        # どれか失敗で全部ロールバック
end
```
- 楽観ロック（`lock_version` カラム）/ 悲観ロック（`with_lock` / `lock!`）。

## Rails 4 のDB関連トピック
- **`where.not` / `find_or_create_by` / `find_by`** の整備。
- **`enum`（4.1〜）**：整数カラムを名前で扱う。
- **`foreign_key: true`（4.2〜）**：マイグレーションで外部キー制約。
- （`encrypts`・`load_async`・複合主キーは7系。Rails 4 には無い。）

## ハマりどころ / アンチパターン
- **`scope` は lambda 必須**（Rails 3 記法は不可）。
- **N+1**（最頻）→ `includes`。
- **`save` と `save!`**：前者は失敗を `false` で返すだけ。握り潰し注意。
- **`default_scope`** の副作用 → 予期せぬ絞り込み。基本使わない。
- **`update_all` / `delete_all`** はバリデーション・コールバックを通さない（`insert_all`/`upsert_all` は7系で、4には無い）。
- **time zone**：`Time.now` でなく `Time.current` / `Time.zone.now`。`config.active_record.time_zone_aware_attributes` を確認。
- マイグレーションとモデルの不整合（カラム消したのにコード参照が残る）。

## ActiveRecord 特有の現象と対策（N+1以外）
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| `count`/`size`/`length` の取り違え | `count`=毎回COUNT、`size`=ロード済みなら配列長、`length`=必ず全件ロード | 一覧後の件数は `size`、件数だけ `count` |
| 関連件数で COUNT 乱発 | `post.comments.count` を一覧ループで | **`counter_cache`** で1カラム参照に |
| 大量データでメモリ枯渇 | `Post.all.each` は全件展開 | **`find_each` / `find_in_batches`** |
| 不要な全カラム取得 | `Post.all` は全列でモデル生成 | **`pluck` / `select`** |
| 存在確認が重い | `present?` は全件ロード | **`exists?`** |
| コネクションプール枯渇 | Webサーバのスレッド/プロセス数と `pool` 不一致 | `pool` ≥ 同時実行数 |

## 関連
[model.md](./model.md) / [controller.md](./controller.md) / [active_job.md](./active_job.md)
