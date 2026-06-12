# Concern（Rails 7）

## ひとことで言うと
**複数のモデル/コントローラに共通する振る舞いを切り出してミックスインするための仕組み**。`ActiveSupport::Concern` を使ったモジュールで、`app/models/concerns/` や `app/controllers/concerns/` に置く。

## 役割・なぜ必要か
- 「論理削除できる」「slugを持つ」「タイムスタンプで絞り込める」など、**複数クラスで同じ振る舞いを使い回したい**ときに、コピペせず1箇所にまとめるためにある（DRY）。
- 素のRubyモジュールだと「`included` フックの中で `has_many` を呼びたい」「クラスメソッドも一緒に生やしたい」が書きづらい。`ActiveSupport::Concern` はその定型を `included do ... end` / `class_methods do ... end` という形で**きれいに書けるようにする糖衣**。
- モデルが太ってきたときの分割先のひとつ（責務単位の切り出し）。→ [model.md](./model.md)

## 基本の書き方（コード）
```ruby
# app/models/concerns/archivable.rb
module Archivable
  extend ActiveSupport::Concern

  # include された側のクラス文脈で実行される（関連・スコープ・コールバック等）
  included do
    scope :archived,  -> { where.not(archived_at: nil) }
    scope :active,    -> { where(archived_at: nil) }
  end

  # インスタンスメソッド（include したクラスのインスタンスに生える）
  def archive!
    update!(archived_at: Time.current)
  end

  def archived?
    archived_at.present?
  end

  # クラスメソッド（Model.xxx で呼べる）
  class_methods do
    def archive_all_before(time)
      active.where("created_at < ?", time).update_all(archived_at: Time.current)
    end
  end
end
```
```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Archivable   # これだけで archive! / archived? / scope が使える
end
```

## 実務での使い方・定番パターン
- **横断的な振る舞いを名詞 + able で命名**：`Archivable` / `Sluggable` / `Searchable` / `Trackable` など、「何ができるか」が名前で分かるように。
- **コントローラ側のConcern**：`before_action` で使う共通処理（例 `Authenticatable` で `authenticate_user!`）を `app/controllers/concerns/` に置く。→ [controller.md](./controller.md)
- **`included do` の中**にはクラスマクロ（`has_many` `validates` `scope` `before_save` など）、**メソッド定義は外**、**クラスメソッドは `class_methods do`**、と置き場所を固定すると読みやすい。
- 共通化の前にまず「本当に複数クラスで使うか」を確認。1クラスでしか使わないなら普通にモデルに書く（YAGNI）。

## ハマりどころ / アンチパターン
- **何でも詰め込んだ巨大Concern**：`Sluggable` のはずが検索もエクスポートも入っている、という状態は密結合の温床。**1Concern = 1責務**で小さく保つ。
- **「共通っぽいから」で早すぎる抽象化**：2箇所で似ているだけで括ると、後で片方の都合が変わって破綻する。重複が実際に痛くなってから括る。
- **Concern同士の暗黙の依存**：あるConcernが別のConcernのメソッドを前提にする等。依存は明示するか、設計を見直す。
- **手続き的なユースケース（複数モデルにまたがる処理）をConcernに押し込む**：それは Service の領分。Concernは「クラスに生やす属性的な振る舞い」向き。→ [service_form.md](./service_form.md)

## 関連
[model.md](./model.md) / [service_form.md](./service_form.md)
