# アクション（Action）（Hanami 2）

## ひとことで言うと
Hanami の「コントローラ」相当。ただし Rails と違い **1アクション = 1クラス**。`app/actions/books/index.rb` に `class Books::Index < MyApp::Action` を1つだけ置き、HTTPリクエスト1種類を処理する。

## 役割・なぜ必要か
- ルーティングからの1エンドポイント（例: `GET /books`）を受け、レスポンスを組み立てる責務だけを持つ。
- Rails の `BooksController` に `index/show/create...` を詰め込む形ではなく、**振る舞いごとにクラスを分ける**。1クラス1責務になり、依存も最小化できる。
- ロジック本体（DB操作・業務処理）はアクションに書かず、**DI で注入したコンポーネント**に委譲する。アクションは「入力を受け取り→処理に渡し→レスポンスを返す」薄い層に保つ。→ [dependency_injection.md](./dependency_injection.md)
- ビューと**分離**されている。アクションはHTTPの入出力を担い、HTML生成はビュー側。→ [views_templates.md](./views_templates.md)

## 基本の書き方（コード）
```ruby
# app/actions/books/index.rb
module MyApp
  module Actions
    module Books
      class Index < MyApp::Action
        # 依存を注入（コンテナのキー = ファイルパス由来）
        include Deps["repositories.book_repo"]

        def handle(request, response)
          # クエリ文字列やボディは request.params で取る
          page = request.params[:page] || 1
          books = book_repo.listing(page: page)

          # レスポンスは response に代入して組み立てる（return ではない）
          response.body = books.to_json
        end
      end
    end
  end
end
```

```ruby
# app/actions/books/show.rb
module MyApp
  module Actions
    module Books
      class Show < MyApp::Action
        include Deps["repositories.book_repo"]

        def handle(request, response)
          book = book_repo.find(request.params[:id])

          unless book
            response.status = 404           # ステータスは数値 or シンボル
            response.body = { error: "not found" }.to_json
            return
          end

          response.status = 200
          response.format = :json
          response.body = book.to_json
        end
      end
    end
  end
end
```

## 実務での使い方・定番パターン
- **`#handle(request, response)` が唯一のエントリポイント**。`request` から入力、`response` に出力を書く。値を `return` してもレスポンスにはならない（`response` を直接いじる）。
- **入力取得は `request.params`**。クエリ・ルートパラメータ・ボディがマージされて入る。生の値は信用せず、**Contract で検証**するのが定番。アクション内に `params do ... end` を書くか、別 Contract に切り出す。→ [validation.md](./validation.md)
- **レスポンス組み立て**：`response.body=`（本体）、`response.status=`（`200` / `:not_found` 等）、`response.format=`（`:json` / `:html`）、`response.headers[...]=`。
- **ビューに繋ぐ場合**は `include Deps["views.books.index"]` のようにビューを注入し、`response.render(view, **exposures)` で描画する（HTMLアプリ）。→ [views_templates.md](./views_templates.md)
- **before/after コールバック**：`before :authenticate` のように共通処理を差し込める。認証など横断的処理はベース `MyApp::Action` 側に置くと使い回せる。
- **依存はコンストラクタ注入**。`include Deps["..."]` した時点でそのキーがインスタンスメソッド名（パス末尾）として使える。テスト時は `.new(book_repo: fake)` で差し替え可能。

## ハマりどころ / アンチパターン
- **Rails 感覚の「1コントローラ多アクション」で書こうとする**のが最大の落とし穴。Hanami は **1ファイル1クラス1アクション**。`Books::Index` と `Books::Show` は別クラス・別ファイル。
- **`handle` のシグネチャを間違える**：必ず `def handle(request, response)`。戻り値ではなく `response` への代入で結果を返す。`response.body = ...` を忘れると空レスポンスになる。
- **`params` を無検証で使う**：型・必須チェックを Contract に通さず `request.params[:id]` を直接DBへ渡すと壊れやすい。境界で検証する。→ [validation.md](./validation.md)
- **アクションに業務ロジックを詰める**：Fat Action 化すると Rails の Fat Controller と同じ轍。DBアクセスやドメイン処理は注入したコンポーネントへ寄せ、アクションは薄く保つ。
- **アクションでHTMLを組み立てる**：表示はビュー＋テンプレートの責務。アクションで文字列連結してHTMLを返さない。→ [views_templates.md](./views_templates.md)
- **ルーティングとのクラス名ズレ**：`config/routes.rb` の `to:` 指定（例 `to: "books.index"`）とクラスのパスが一致しないと解決できない。→ [routing.md](./routing.md)

## 関連
[routing.md](./routing.md) / [dependency_injection.md](./dependency_injection.md) / [views_templates.md](./views_templates.md) / [validation.md](./validation.md)
