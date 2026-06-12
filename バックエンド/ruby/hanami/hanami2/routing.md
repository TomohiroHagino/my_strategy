# ルーティング（Hanami 2）

## ひとことで言うと
受け取ったHTTPリクエスト（メソッド＋パス）を**どのアクションに渡すか**を定義する仕組み。設定は **`config/routes.rb`** に集約され、`to:` には**アクションのコンテナキー文字列**（例 `"books.index"`）を書く。

## 役割・なぜ必要か
- Hanami 2 のアクションは「1アクション＝1クラス」。ルーターはパスとそのクラスを繋ぐ唯一の場所で、`to:` の文字列がそのままコンテナ上のアクションキーを指す。
- `"books.index"` → `Bookshelf::Actions::Books::Index`（コンテナキー `"actions.books.index"`）のように、**文字列がクラス解決に直結**する。Rails の `controller#action` 文字列に似ているが、指しているのはコントローラのメソッドではなく**独立したアクションクラス**。
- ルートを1ファイルに集約することで、アプリの入口（公開しているエンドポイント）が一覧で読める。

## 基本の書き方（コード）
```ruby
# config/routes.rb
module Bookshelf
  class Routes < Hanami::Routes
    # ルートパス → Home::Show アクション
    root to: "home.show"

    # GET /books → Actions::Books::Index
    get "/books", to: "books.index"

    # GET /books/:id → Actions::Books::Show（params[:id] で受け取る）
    get "/books/:id", to: "books.show"

    # POST /books → Actions::Books::Create
    post "/books", to: "books.create"

    # 名前空間でまとめる（/admin/... 配下）
    scope "admin" do
      get "/books", to: "admin.books.index"
    end
  end
end
```

```ruby
# to: の文字列 "books.index" が解決される先
# app/actions/books/index.rb
module Bookshelf
  module Actions
    module Books
      class Index < Bookshelf::Action
        def handle(request, response)
          response.body = "list of books"
        end
      end
    end
  end
end
```

```ruby
# slice ごとのルート（slices/admin/config/routes.rb 等で
# slice 側のアクションへ向ける。詳細は slices.md 参照）
# config/routes.rb 側で slice をマウントする書き方もある
slice :admin, at: "/admin" do
  get "/books", to: "books.index"   # → Admin::Actions::Books::Index
end
```

## 実務での使い方・定番パターン
- **`to:` はアクションのキー**。`"books.index"` のドットは名前空間の区切りで、`Actions` を起点に `Books::Index` を解決する。パスのネストとクラスのネストを一致させると読みやすい。→ [actions.md](./actions.md)
- **`root to:`** でトップページを定義。値は他と同じくアクションキー（`"home.show"` など）。
- **`scope` / `slice`** で URL プレフィックスと名前空間をまとめる。管理画面など機能群が分かれたら slice 単位でルートを持たせ、メインの routes.rb からマウントする。→ [slices.md](./slices.md)
- パラメータは `:id` のように書き、アクション側で `request.params[:id]` として参照する。検証は dry-validation の Contract に任せる。→ [validation.md](./validation.md)
- ルートを変えたら `hanami console` や起動ログで意図通りアクションが解決されるか確認する。`to:` の文字列ミスは起動・リクエスト時に表面化する。

## ハマりどころ / アンチパターン
- **`to:` の文字列はメソッド名ではない**。Rails の `"books#index"`（コントローラ#アクション）と混同しがちだが、Hanami は `"books.index"`（ドット区切り）で、指す先は**独立したアクションクラス** `Books::Index`。`#` ではなく `.`。
- **文字列とクラスのズレ**。`"books.index"` と書いたのにファイルが `app/actions/book/index.rb`（単数）だった、名前空間が欠けている等で解決に失敗する。**ドットの各セグメント＝モジュールのネスト**と機械的に対応させる。
- **slice のルートをメイン側に書いてしまう**。slice 固有のエンドポイントは slice のコンテナ内アクションを指す必要がある。どのコンテナの `Actions` を解決するかを意識する（メイン routes か slice routes か）。
- **大量のルートを1ファイルに雑然と並べる**。機能が増えたら `scope` / `slice` で構造化しないと、入口一覧の可読性という利点が失われる。
- パスパラメータの**型・存在チェックをルーターに期待しない**。ルーターは振り分けのみ。値の検証はアクション内の Contract で行う。

## 関連
[actions.md](./actions.md) / [slices.md](./slices.md) / [project_structure.md](./project_structure.md)
