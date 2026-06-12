# Concern（Rails 5）

## ひとことで言うと
複数のモデルやコントローラで**共通する振る舞いを切り出して再利用するためのモジュール**。`ActiveSupport::Concern` を `extend` して書く。

## 役割・なぜ必要か
- 「論理削除」「公開フラグ管理」「タイムスタンプ整形」など、複数クラスにまたがる共通の機能を、継承ツリーをいじらずに後付けで混ぜ込む（mixin）ためにある。
- Fat Model / Fat Controller を分割する手段の1つ。関連する関連定義・スコープ・メソッドをひとまとめにできる。

## 基本の書き方（コード）
```ruby
# app/models/concerns/publishable.rb
module Publishable
  extend ActiveSupport::Concern

  included do                              # include された側のクラス文脈で実行される
    scope :published, -> { where(published: true) }
    validates :published, inclusion: { in: [true, false] }
  end

  class_methods do                         # クラスメソッドを定義
    def publish_all!
      update_all(published: true)
    end
  end

  def publish!                             # インスタンスメソッド
    update!(published: true)
  end
end
```
```ruby
# 使う側
class Post < ApplicationRecord
  include Publishable                      # post.publish! / Post.published が使える
end
```
- コントローラ用は `app/controllers/concerns/` に置き、`before_action` 等を `included do` に書く。

## 実務での使い方・定番パターン
- **モデル横断の振る舞い**：`Publishable` / `SoftDeletable` / `Searchable` など名詞＋able で命名するのが慣習。
- **コントローラ共通処理**：認証・ページネーション・例外ハンドリングを Concern に切り出して複数コントローラで `include`。→ [filters.md](./filters.md)
- **`included do`** に関連・スコープ・コールバック、**`class_methods do`** にクラスメソッド、本体にインスタンスメソッド、と置き場所を分ける。
- Rails 5 では `app/models/concerns` と `app/controllers/concerns` が**自動で読み込みパスに入る**ので、そこに置けばそのまま使える。

## ハマりどころ / アンチパターン
- **何でも Concern に押し込む**：分割しただけで結合度が下がらないと「ファイルが増えただけ」。本当に複数クラスで共有するものに限る。
- **Concern 同士の依存**：A が B のメソッドを前提にする等、暗黙の依存が増えると追えなくなる。独立させる。
- **状態の共有を期待する**：Concern はあくまで振る舞いの mixin。複数モデルにまたがる「手続き」は **Service オブジェクト**の方が適切な場合が多い。→ [service_form.md](./service_form.md)
- **命名衝突**：複数 Concern が同名メソッドを定義すると後勝ちで上書きされる。

## 関連
[model.md](./model.md) / [service_form.md](./service_form.md) / [filters.md](./filters.md)
