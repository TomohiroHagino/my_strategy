# ビュー / テンプレート（View / Template）（Hanami 2）

## ひとことで言うと
Hanami のHTML生成層。**ビューは独立したクラス**（`app/views/books/index.rb`）で、対応する**テンプレート**（`app/templates/books/index.html.erb`）へ **exposures** でデータを渡して描画する。Rails のように「コントローラがインスタンス変数でビューにデータを渡す」構造ではない。

## 役割・なぜ必要か
- **アクションとビューを分離**するのが Hanami の思想。アクションはHTTP入出力、ビューは「何をテンプレートに渡し、どう整形するか」を担う。責務が分かれ、ビュー単体でテスト可能になる。
- ビュー = `Hanami::View` を継承したクラス。**exposures**（`expose :books`）でテンプレートに渡す値を宣言的に定義する。Rails の `@books` のような暗黙の共有変数ではなく、「このテンプレートは何を必要とするか」が明示される。
- 表示用の整形ロジック（日付フォーマット、表示名など）をビュー／パーシャル側に集約でき、テンプレートを薄く保てる。

## 基本の書き方（コード）
```ruby
# app/views/books/index.rb
module MyApp
  module Views
    module Books
      class Index < MyApp::View
        # ビューも依存注入できる（コンテナのキー = パス由来）
        include Deps["repositories.book_repo"]

        # exposures: テンプレートに渡す値を宣言
        # ブロック引数はアクションから render(...) で渡した値
        expose :books do |page: 1|
          book_repo.listing(page: page)
        end

        # 別の exposure を参照することもできる（依存順は解決される）
        expose :count do |books:|
          books.size
        end
      end
    end
  end
end
```

```erb
<%# app/templates/books/index.html.erb %>
<h1>Books (<%= count %>)</h1>
<ul>
  <% books.each do |book| %>
    <li><%= book.title %></li>
  <% end %>
</ul>
```

```ruby
# app/actions/books/index.rb（アクション側からビューを呼ぶ）
module MyApp
  module Actions
    module Books
      class Index < MyApp::Action
        include Deps[view: "views.books.index"]

        def handle(request, response)
          # exposure のブロック引数として渡る
          response.render(view, page: request.params[:page] || 1)
        end
      end
    end
  end
end
```

## 実務での使い方・定番パターン
- **exposures がデータ受け渡しの中心**。`expose :name do |args| ... end` の戻り値が、テンプレート内で `name` として参照できる。ブロックなしの `expose :name` は `render(view, name: ...)` の入力をそのまま通す。
- **exposure 同士の依存**：あるexposureのブロック引数に別exposure名を書くと、Hanami が解決順を組んで渡してくれる（上例の `count` が `books` を受け取る形）。
- **private exposure**：`private_expose :raw` はテンプレートに出さず、他のexposureからのみ使える中間値に使う。
- **レイアウト**：`app/templates/layouts/app.html.erb` が共通の外枠。`<%= yield %>` 相当でビューのテンプレートが差し込まれる。ビュークラスで `config.layout` を指定。
- **パーシャル**：`<%= render partial: "books/book", book: book %>` のように部分テンプレート（`_book.html.erb`）を切り出して再利用。
- **ヘルパー / Part**：表示整形は `Hanami::View::Part`（モデルをラップして表示用メソッドを足す）に寄せると、テンプレートをロジックレスに保てる。
- **アクションからの呼び出し**は `response.render(view, **exposure_inputs)`。HTMLアプリではこれが基本形。→ [actions.md](./actions.md)

## ハマりどころ / アンチパターン
- **Rails の「コントローラで `@books` を作ってビューで使う」発想で書く**：Hanami にその暗黙共有はない。**アクション → `render(view, key: ...)` → exposure のブロック引数** という明示的な流れに従う。
- **アクションとビューの責務を混ぜる**：アクション内でHTML文字列を組んだり、ビュー内でHTTPステータスをいじったりしない。分離が前提。→ [actions.md](./actions.md)
- **exposure 名とテンプレートの参照名のズレ**：`expose :books` なのにテンプレートで `@books` や `book_list` と書くと未定義。宣言した名前そのままで参照する。
- **テンプレートにロジックを書きすぎる**：条件分岐や整形を ERB に詰め込むと読めなくなる。整形は Part / ヘルパー、データ準備は exposure へ。
- **パス規約のズレ**：ビュークラス `Views::Books::Index` ↔ テンプレート `templates/books/index.html.erb` の対応が崩れるとテンプレートが見つからない。
- **exposure のブロック引数を必須にしてしまい render 時に渡し忘れる**：デフォルト値（`|page: 1|`）を付けるか、render 側で必ず渡す。

## 関連
[actions.md](./actions.md) / [dependency_injection.md](./dependency_injection.md) / [routing.md](./routing.md)
