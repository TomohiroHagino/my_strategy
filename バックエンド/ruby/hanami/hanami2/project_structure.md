# プロジェクト構成（Hanami 2）

## ひとことで言うと
Hanami 2 アプリの**ディレクトリの分け方とその意味**。中心は **`app/`（メインアプリ）** ＋ **`slices/`（サブアプリ）** ＋ `config/`（設定）＋ `lib/`（共有コード）。`app/` 配下のクラスは**自動でDIコンテナに登録**されるのが最大の特徴。

## 役割・なぜ必要か
- Hanami 2 は「規約でDBと繋ぐ」のではなく「**コンポーネントをコンテナに集め、依存を注入する**」設計。だからファイルの置き場所＝コンテナ上のキーになり、構成そのものが配線図になる。
- `app/` はアプリの本体（アクション・ビュー・オペレーション等）。`slices/` は「ある程度独立した機能のかたまり」を別アプリのように切り出す場所。最初は `app/` だけで始め、育ったら `slices/` に分割する。
- `lib/` はフレームワークに依存しない純粋なドメインコードや共有ユーティリティの置き場。`config/` はルート・設定・アプリ定義をまとめる。

## 基本の書き方（コード）
```bash
bookshelf/
├── app/                      # メインアプリ（= "bookshelf" コンテナ）
│   ├── action.rb             # アクションの基底クラス
│   ├── view.rb               # ビューの基底クラス
│   ├── actions/
│   │   └── books/
│   │       └── index.rb      # → Bookshelf::Actions::Books::Index
│   ├── views/
│   │   └── books/
│   │       └── index.rb      # → Bookshelf::Views::Books::Index
│   ├── templates/            # ビューのテンプレート(.html.erb 等)
│   └── operations/           # ドメイン操作(任意の置き場)
├── config/
│   ├── app.rb                # アプリ本体の定義（Bookshelf::App）
│   ├── routes.rb             # ルーティング
│   └── settings.rb           # 設定スキーマ
├── lib/
│   └── bookshelf/            # 共有コード（トップレベル名前空間）
├── slices/
│   └── admin/                # サブアプリ（= "admin" コンテナ）
│       ├── actions/
│       └── views/
└── spec/
```

```ruby
# config/app.rb — アプリの定義。これがトップレベル名前空間を決める
require "hanami"

module Bookshelf
  class App < Hanami::App
  end
end
```

```ruby
# app/actions/books/index.rb
# パス app/actions/books/index.rb → 定数 Bookshelf::Actions::Books::Index
# コンテナキーは "actions.books.index"（app/ の階層がキーになる）
module Bookshelf
  module Actions
    module Books
      class Index < Bookshelf::Action
        def handle(request, response)
          response.body = "books"
        end
      end
    end
  end
end
```

## 実務での使い方・定番パターン
- **`app/` 配下＝コンテナ登録**。`app/operations/create_book.rb` を置けば `"operations.create_book"` というキーで解決でき、他のクラスから `include Deps["operations.create_book"]` で注入できる。置き場所＝配線。→ [dependency_injection.md](./dependency_injection.md)
- **`lib/` はコンテナ外の純ドメイン**。フレームワーク非依存のロジック・値オブジェクトはここ。`app/` のコンポーネントから普通に `require` / 参照して使う。
- **`slices/` は機能の独立境界**。管理画面 (`admin`)、API (`api`) のように責務が分かれたら slice に切る。各 slice は独自のコンテナ・名前空間・ルートを持つ。→ [slices.md](./slices.md)
- **`config/` は1か所**。ルート（routes.rb）、設定スキーマ（settings.rb）、アプリ定義（app.rb）が集約され、ここを読めばアプリの骨格が分かる。
- 命名は **Zeitwerk** 準拠。`app/views/books/index.rb` は `Bookshelf::Views::Books::Index` でなければロードされない。

## ハマりどころ / アンチパターン
- **`app/` と `slices/` の境界を曖昧にする**。「とりあえず全部 app/」だと slice の独立性メリットが消え、「最初から細かく slice 分割」だと過剰設計になる。**app/ で始めて、独立が見えたら slice へ**が定石。
- **コンテナ登録の規約を無視した置き場所**。`app/` 配下に置けば登録されるが、キーはパス階層から決まる。期待したキーで解決できない時は、まず `Hanami.app.keys` で実際の登録キーを確認する。
- **命名とパスのズレ**。`uninitialized constant` の大半はこれ。`app/actions/books/index.rb` に `class Index`（名前空間なし）と書く、複数形/単数形を間違える等。パスの各セグメント＝モジュールのネスト、と機械的に対応させる。
- **`lib/` のコードに `Deps[]` を期待する**。`lib/` はコンテナ外なので自動注入の対象外。注入を使いたいものは `app/`（または slice）配下へ置く。
- **モデル層をRails風に探す**。`app/models/` は無い。データは ROM の Relation / Repository が担当する。→ [persistence_rom.md](./persistence_rom.md)

## 関連
[slices.md](./slices.md) / [dependency_injection.md](./dependency_injection.md) / [getting_started.md](./getting_started.md)
