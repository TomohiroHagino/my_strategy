# Active Record（Rails 6）

## ひとことで言うと
DBのテーブルとRubyのクラス/オブジェクトを対応づけ、マイグレーション・クエリ・関連・トランザクションをRubyで書けるO/Rマッパー。Rails 6では複数DB・insert_all・6.1のstrict_loadingが加わった。

## 役割・なぜ必要か
SQLを直接書かずにテーブル定義（マイグレーション）とデータ操作（クエリ）を行う。`Model < ApplicationRecord` でテーブル名・カラムが自動マッピングされ、関連・バリデーション・コールバックも一元管理できる。

## 基本の書き方（コード）
マイグレーション（`db/migrate/xxxx_create_users.rb`）:
```ruby
class CreateUsers < ActiveRecord::Migration[6.1]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email
      t.references :company, foreign_key: true
      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```
```bash
rails db:migrate        # 適用（schema.rb 更新）
rails db:rollback        # 直前を巻き戻し
rails db:migrate:status  # 状態確認
```
`change` で書けない複雑な変更は `up`/`down`、またはブロックで `reversible` を使う:
```ruby
def change
  reversible do |dir|
    dir.up   { execute "UPDATE users SET name = 'unknown' WHERE name IS NULL" }
    dir.down { } # 巻き戻し時の処理
  end
  change_column_null :users, :name, false
end
```

## 実務での使い方・定番パターン
クエリ:
```ruby
User.where(active: true).order(created_at: :desc).limit(10)
User.find(1)                 # 主キー。無ければ RecordNotFound
User.find_by(email: "a@b.c") # 無ければ nil
User.where(active: true).pluck(:id, :name)  # 配列で取得
User.pick(:id, :name)        # 6.0+。最初の1件のカラム値（limit(1).pluck相当）

class User < ApplicationRecord
  scope :active, -> { where(active: true) }
  scope :recent, ->(n) { order(created_at: :desc).limit(n) }
end
User.active.recent(5)
```
関連とJOIN（N+1対策の核心）:
```ruby
class User < ApplicationRecord
  belongs_to :company
  has_many :posts
end

User.includes(:posts).each { |u| u.posts.size }  # まとめてロード（N+1回避）
User.preload(:posts)   # 必ず別クエリ
User.eager_load(:posts) # 必ずLEFT JOIN（whereで関連を絞る時）
User.joins(:posts).where(posts: { published: true }) # INNER JOIN、関連で絞り込み
```
N+1問題: `User.all.each { |u| u.company.name }` は会社をN回引く。`includes(:company)` で1回にまとめる。
6.1の strict_loading（N+1をエラーで検出）:
```ruby
user = User.strict_loading.find(1)
user.posts  # => ActiveRecord::StrictLoadingViolationError
User.strict_loading.all          # モデル単位で強制
# config/application.rb
config.active_record.strict_loading_by_default = true # アプリ全体で既定ON
```
バルク挿入（6.0+。**バリデーション・コールバックは走らない**）:
```ruby
User.insert_all([{ name: "a" }, { name: "b" }])   # 既存衝突はスキップ
User.insert_all!([...])                            # 衝突で例外
User.upsert_all([{ id: 1, name: "x" }])            # あればUPDATE
```
6.1の where.missing（関連レコードが無いものを取得）:
```ruby
Post.where.missing(:author)  # author が無い post。LEFT JOIN + IS NULL
```
delegated_types（6.1）: STIの代替として、共通属性を1テーブル＋型別テーブルに委譲する仕組み。詳しくは公式参照。
トランザクションとロック:
```ruby
ActiveRecord::Base.transaction do
  account.update!(balance: 0)
  log.save!
end  # 例外で全ロールバック
user = User.lock.find(1)           # 悲観ロック（SELECT ... FOR UPDATE）
user.with_lock { user.update!(...) }
```
複数DB（読み書き分離・用途別分割）: `database.yml` の primary/replica、`connects_to`、`connected_to(role: :reading) { ... }` で切り替える。詳細は [multiple_databases.md](./multiple_databases.md) へ。

## ハマりどころ / アンチパターン
- N+1: ループ内で関連を引いていないか。`includes`/`preload` を使う。検出は6.1の `strict_loading` か bullet gem。
- `insert_all`/`upsert_all` は**バリデーションもコールバックも実行されない**。整合性は呼び出し側で担保する。
- `joins` + `where` で関連を絞ると重複行が出る。件数取得は `distinct` か `eager_load`。
- マイグレーション忘れ（`Migrations are pending`）→ `rails db:migrate`。`schema.rb` をコミットし忘れない。
- 複数DBで読み取り直後の書き込み/書き込み直後の読み取りはレプリカ遅延に注意（[multiple_databases.md](./multiple_databases.md)）。
- `find` は無いと例外、`find_by` は nil。混同して `NoMethodError on nil` を出さない。

## 関連
[model.md](./model.md) / [multiple_databases.md](./multiple_databases.md) / [console.md](./console.md) / [testing.md](./testing.md) / [zeitwerk.md](./zeitwerk.md) / [pitfalls.md](./pitfalls.md)
