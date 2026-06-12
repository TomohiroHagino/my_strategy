# module（名前空間・ミックスイン）（Rails 5）

## ひとことで言うと
Ruby の `module` を Rails では主に2用途で使う：**名前空間**（`Admin::UsersController` のようにクラスをグループ化して衝突を防ぐ）と **ミックスイン**（共通処理を `include`/`extend` で複数クラスに混ぜ込む）。機能を丸ごと切り出す **Engine** も module の応用。

## 役割・なぜ必要か
- **名前空間**：管理画面・API・バージョンなどで、同名クラスの衝突を避けて整理する（`Admin::User` と `User` は別物）。
- **ミックスイン**：継承は1つしかできないが、module なら複数の振る舞いを横断的に足せる。
- Rails は「フォルダ構造＝モジュールの入れ子」という規約なので、`module` を強く意識せずとも名前空間が自然に作られる。

## 名前空間（namespace）

### フォルダ＝モジュール
```
app/controllers/admin/users_controller.rb  →  Admin::UsersController
app/models/billing/invoice.rb              →  Billing::Invoice
```
```ruby
# app/controllers/admin/users_controller.rb
# classic autoloader ではネスト記法のほうが安全
module Admin
  class UsersController < ApplicationController
  end
end
```

### ルーティングの namespace / scope
```ruby
# config/routes.rb
namespace :admin do
  resources :users        # URL /admin/users → Admin::UsersController
end

scope "/admin" do
  resources :users        # URLだけ admin、クラスは UsersController
end

scope module: :admin do
  resources :users        # URLは /users、クラスは Admin::UsersController
end
```

## ミックスイン（mixin）
```ruby
module Greetable
  def greet
    "Hi, #{name}"
  end
end

class User
  include Greetable   # インスタンスメソッドに（user.greet）
end

class Robot
  extend Greetable    # クラスメソッドに（Robot.greet）
end
```
- `include` … インスタンスメソッドへ。`extend` … クラス（特異）メソッドへ。`prepend` … 既存メソッドの手前に割り込む。
- Rails では `ActiveSupport::Concern` を使うと `included do ... end` や依存解決が書きやすい。→ [concern.md](./concern.md)

## 自動読み込みとの関係（classic autoloader）
Rails 5 は **classic オートローダー**（Zeitwerk は Rails 6 から）。定数が未定義のとき、命名規約からファイルパスを推測して読み込む。
- フォルダ＝モジュールの対応（`admin/users_controller.rb` → `Admin::UsersController`）は同じ。
- ただし Zeitwerk ほど厳密でなく、名前空間絡みで定数が見つからない時に `require_dependency "admin/users_controller"` が要る場面がある。
- 開発環境のクラス再読み込みで「A copy of X has been removed from the module tree but is still active」系の定数衝突に遭遇しやすい。→ Rails 6 の Zeitwerk で厳密化・解消された。
- 名前空間クラスは `class Admin::UsersController` より、`module Admin; class UsersController` の**ネスト記法**のほうが安全。

## Engine（module で機能を切り出す）
```ruby
module MyBlog
  class Engine < ::Rails::Engine
    isolate_namespace MyBlog   # MyBlog:: で隔離された小さなRailsアプリ
  end
end
```
- 認証・管理画面など再利用したい機能を gem（Engine）として分離できる。Devise なども Engine。

## 実務での使い方・定番パターン
- 管理画面は `Admin::`、APIは `Api::V1::` で名前空間を切り、フォルダも対応させる。
- 共通の before_action や scope は Concern（module）に切り出して複数のコントローラ/モデルで `include`。
- 肥大化した機能は Engine 化して別gemへ。

## ハマりどころ / アンチパターン
- **classic autoloader での定数未検出**：名前空間絡みで `uninitialized constant` が出たら `require_dependency` を検討。
- **include と extend の取り違え**：インスタンス側かクラス側か。
- **namespace と `scope module:` の混同**：URL とクラス名のどちらに `admin` を付けたいかで選ぶ。
- 名前空間を深くしすぎると `Foo::Bar::Baz::Qux` で可読性が落ちる。

## 関連
[concern.md](./concern.md) / [routing.md](./routing.md) / [controller.md](./controller.md)
