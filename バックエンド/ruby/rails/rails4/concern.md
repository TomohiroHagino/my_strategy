# Concern（Rails 4）

## ひとことで言うと
複数のモデルやコントローラで共通する振る舞いを、`ActiveSupport::Concern` を使った**モジュールに切り出して `include` する仕組み**。`app/models/concerns/` `app/controllers/concerns/` に置く。

## 役割・なぜ必要か
- 同じバリデーション・スコープ・メソッド群を複数モデルで使い回すために、横断的な機能をモジュール化する。
- Fat Model を機能単位に分割して見通しを良くする。`included do ... end` でクラスマクロ（`validates` 等）も含められる点が普通のモジュールと違う。
- （Rails 4 で `app/models/concerns` `app/controllers/concerns` が**標準ディレクトリとして用意された**。）

## 基本の書き方（コード）
```ruby
# app/models/concerns/archivable.rb
module Archivable
  extend ActiveSupport::Concern

  included do
    scope :archived,   -> { where.not(archived_at: nil) }
    scope :unarchived, -> { where(archived_at: nil) }
  end

  def archive!
    update!(archived_at: Time.current)
  end

  class_methods do          # クラスメソッドを定義（4で class_methods ブロックが使える）
    def archive_all_before(time)
      where("created_at < ?", time).update_all(archived_at: Time.current)
    end
  end
end
```
```ruby
# app/models/post.rb
class Post < ActiveRecord::Base
  include Archivable
end

Post.archived          # スコープが使える
post.archive!          # インスタンスメソッド
Post.archive_all_before(1.year.ago)  # クラスメソッド
```

## 実務での使い方・定番パターン
- **モデル横断の振る舞い**：論理削除（archivable）、状態フラグ、共通スコープ等。
- **コントローラ横断の処理**：`app/controllers/concerns/` に認証ヘルパーや共通レスポンスを切り出し、各コントローラで `include`。
- **`included do ... end`** に `validates` / `belongs_to` / `before_save` などクラスマクロを置ける。
- **`class_methods do ... end`** でクラスメソッドを定義（`module ClassMethods` を手書きする旧来形の糖衣構文）。
- 機能名で命名（`Searchable` / `Taggable` / `SoftDeletable`）。

## ハマりどころ / アンチパターン
- **何でもConcernに詰める**：複数モデルにまたがる「手続き」は Concern より **Service** が適切。Concern は「モデルが持つべき振る舞い」に限る。→ [service_form.md](./service_form.md)
- **Concern 同士の依存**：A が B に依存…と絡むと追えなくなる。独立性を保つ。
- **`extend ActiveSupport::Concern` 忘れ**：`included do` が使えずエラー。
- **状態（インスタンス変数）を Concern に持たせる**：複数 include で衝突。状態より振る舞いを切り出す。
- **共通化の早すぎる抽象化**：2箇所で似ているだけで Concern 化すると、後で差分が出て破綻。重複が本物になってから切り出す（DRYの適用は実需が出てから）。

## 関連
[model.md](./model.md) / [service_form.md](./service_form.md) / [controller.md](./controller.md)
