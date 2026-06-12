# Concern（Rails 6）

## ひとことで言うと
複数のモデルやコントローラに共通するロジックを `ActiveSupport::Concern` を使ってモジュールに切り出し、`include` で取り込む仕組み。

## 役割・なぜ必要か
- 同じメソッドや関連定義を複数クラスに書くと重複する。共通部分をモジュールへ移して `include` する。
- `ActiveSupport::Concern` を使うと、`included do ... end` の中にマクロ（`scope` `validates` `before_save` など）を書け、`class_methods do ... end` でクラスメソッドを定義できる。
- 置き場所は `app/models/concerns` と `app/controllers/concerns`。Rails のオートロード対象。

## 基本の書き方（コード）
```ruby
# app/models/concerns/sluggable.rb
module Sluggable
  extend ActiveSupport::Concern

  included do
    before_save :set_slug
    validates :slug, uniqueness: true, allow_nil: true
  end

  class_methods do
    def find_by_slug!(slug)
      find_by!(slug: slug)
    end
  end

  def set_slug
    self.slug = title.parameterize if slug.blank? && title.present?
  end
end
```
```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Sluggable
end
```
```ruby
# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern

  included do
    before_action :require_login
  end

  private

  def require_login
    redirect_to login_path unless session[:user_id]
  end
end
```

## 実務での使い方・定番パターン
- モデル横断の共通機能（`Sluggable` `SoftDeletable` `Searchable`）を Concern に分離する。
- コントローラ横断の共通処理（ログイン要求、ページネーション設定）を Concern にまとめる。
- 1 Concern = 1 責務。薄く保ち、状態（インスタンス変数）を持たせすぎない。

## ハマりどころ / アンチパターン
- Concern を量産すると、ロジックが分散して `Post` の挙動を追うのに複数ファイルを開く羽目になる。本当に共通化が必要なものだけにする。
- `included do` のマクロは include した順序で評価される。コールバックの実行順に依存するコードは順序に注意。
- Concern に多数のインスタンス変数や状態を持たせると、include 先のクラスと暗黙的に密結合する。メソッド中心に保つ。
- **Zeitwerk** ではファイル名と定数名の一致が必須。`app/models/concerns/sluggable.rb` の定数は `Sluggable`（`Concerns::Sluggable` ではない）。`concerns` ディレクトリは名前空間を作らない。

## 関連
[model.md](./model.md) / [controller.md](./controller.md) / [service_form.md](./service_form.md) / [zeitwerk.md](./zeitwerk.md) / [active_record.md](./active_record.md) / [pitfalls.md](./pitfalls.md)
