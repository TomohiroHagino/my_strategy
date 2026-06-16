# DB / Active Record（Rails 7）

## ひとことで言うと
**Active Record** は、モデルとDBをつなぐ **ORM（オブジェクト関係マッピング）**。テーブルをクラス、行をオブジェクトとして扱い、SQLをほぼ書かずにRubyでDBを操作する。

## 役割・なぜ必要か
- DBアクセスを「Rubyのメソッド呼び出し」に翻訳して、生SQLの重複・ミス・SQLインジェクションを避けるためにある。
- スキーマ管理（マイグレーション）・関連（association）・検証（validation）・クエリ・トランザクションまでを一手に担う。

---

## マイグレーションとは
スキーマ変更を**Rubyで記述しバージョン管理**する仕組み。
```ruby
# db/migrate/2024..._create_posts.rb
class CreatePosts < ActiveRecord::Migration[7.1]
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
- **ハマり**: カラム削除など**不可逆/破壊的変更**は段階移行（まず追加→移行→後で削除）。本番は実行順とロック時間に注意。

## アソシエーション（関連）とは
テーブル間の関係を宣言し、`user.posts` のように辿れるようにする。
```ruby
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
end
class Post < ApplicationRecord
  belongs_to :user            # 7では既定で必須（optional: true で任意）
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
- **乱用注意**：副作用（メール送信・別モデル更新）を詰めると、保存しただけで何が起きるか追えなくなる。重い副作用は **Service / Active Job** へ。

## スコープとは
よく使うクエリに名前を付ける。
```ruby
scope :published, -> { where(published: true) }
scope :recent,    -> { order(created_at: :desc) }
# 使う: Post.published.recent.limit(10)
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
```

## N+1問題とは（最頻の性能バグ）
一覧表示で関連を1件ずつ引き、SQLが `1 + N` 回走る性能劣化。
```ruby
# NG: posts.each { |p| p.user.name } で N+1（posts 1回 + 各userごとに +1）
@posts = Post.includes(:user)   # OK: まとめて読む
```
**対策の選択肢（使い分けが肝）**:
- **`includes(:user)`** … 状況に応じて preload / eager_load を自動選択。まず第一手。
- **`preload(:user)`** … 常に**別クエリ2回**（`SELECT posts` ＋ `SELECT users WHERE id IN (...)`）。JOINしたくない時。
- **`eager_load(:user)`** … **LEFT JOIN で1クエリ**。関連を `where` 条件に使う一覧で有効。
- **`joins(:comments)`** … INNER JOIN で**絞り込みだけ**（関連オブジェクトはロードしない）。件数や存在で絞る用途。
```ruby
# where で関連を条件にするなら eager_load / references が要る（includes 単体だと参照エラー）
Post.includes(:user).where(users: { active: true }).references(:users)
Post.eager_load(:user).where(users: { active: true })   # こちらは references 不要
```
- **検出**：**bullet** gem（不要/不足の eager load を警告）。`strict_loading`（`Post.strict_loading.find ...` やモデルに `self.strict_loading_by_default = true`）で**遅延ロードを例外化して強制検出**。ログのSQL回数も確認。

## トランザクション / ロックとは
```ruby
ActiveRecord::Base.transaction do
  order.save!
  stock.decrement!(:count)        # どれか失敗で全部ロールバック
end
```
- 楽観ロック（`lock_version` カラム）/ 悲観ロック（`with_lock` / `lock!`）。

## Rails 7 のDB関連トピック
- **`encrypts`**：特定カラムをアプリ層で暗号化（`encrypts :ssn`）。
- **`load_async`**：非同期クエリで複数クエリを並行発行し待ち時間短縮。
- 7.1：**複合主キー**サポート。

## ActiveRecord 特有の現象と対策（N+1以外）
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| `count` / `size` / `length` の取り違え | `count` は毎回 `SELECT COUNT` を発行、`size` はロード済みなら配列長・未ロードならCOUNT、`length` は**必ず全件ロード**してから数える | 一覧描画後に件数を見るなら **`size`**。単に件数だけなら `count`。`length` は全件ロード覚悟の時だけ |
| 関連件数で COUNT 乱発 / N+1 | `post.comments.count` を一覧ループで呼ぶと件数クエリがN回 | **`counter_cache`**（`belongs_to :post, counter_cache: true` ＋ `posts.comments_count` カラム）で1カラム参照に |
| 大量データでメモリ枯渇 | `Post.all.each` は全件を一度にメモリ展開 | **`find_each` / `in_batches`**（既定1000件ずつ）で分割ロード |
| 不要な全カラム・全オブジェクト生成 | `Post.all` は全列でモデルを生成 | 必要な値だけなら **`pluck(:id, :name)`**（配列）/ **`select(:id, :name)`**（軽量モデル） |
| `update_all` / `delete_all` で整合崩れ | これらは**バリデーション・コールバック・`updated_at` をスキップ**して直接SQL | 副作用が要るなら1件ずつ `update`/`destroy`。スキップは承知の上で使う |
| `default_scope` の副作用 | 全クエリに暗黙で効き、予期せぬ絞り込み・並び順が混入。外すには `unscoped` が必要 | 基本使わない。絞りは名前付き `scope` で明示的に |
| インデックス欠如で遅い | 外部キー・検索/並び替えカラムに index が無いと全表スキャン | `add_index`（外部キー・`where`/`order` 対象列）。`bin/rails db:migrate` を忘れない |
| 存在確認が重い | `Post.where(...).present?` は全件ロードして判定 | **`exists?`**（`SELECT 1 LIMIT 1` 相当で軽い） |
| `save`/`save!` の握り潰し | `save` は失敗を `false` で返すだけ（例外を投げない） | 失敗を確実に検知したい所は **`save!`**（例外）。`false` を無視しない |
| `insert_all`/`upsert_all` | 一括INSERTだが**バリデーション・コールバックを通さない** | 制約はDB側にも持たせる。アプリ検証が要るなら通常の生成を |
| コネクションプール枯渇 | Puma のスレッド数と `database.yml` の `pool` が不一致だと `ConnectionTimeoutError` | `pool` ≥ スレッド数に合わせる |
| タイムゾーンずれ | `Time.now` はサーバTZ依存 | **`Time.current` / `Time.zone.now`**、DBは UTC 保存 |
| `after_commit` のタイミング | トランザクション内の `after_save` 時点ではまだコミット前 | 外部通知・ジョブ投入は **`after_commit`** で（コミット確定後） |

## 関連
[model.md](./model.md) / [controller.md](./controller.md)
