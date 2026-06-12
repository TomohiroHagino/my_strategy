# DIコンテナ / 依存注入（Deps）（Hanami 2）

## ひとことで言うと
Hanami 2 の**核**。`app/`（や `slices/`）配下のクラスは **DIコンテナに自動登録**され、他のクラスは **`include Deps["相対キー"]`** で依存を注入して使う。バックには dry-system がある。**コンテナのキー = ファイルパス規約**で決まる。

## 役割・なぜ必要か
- コンポーネント（アクション・リポジトリ・サービス等）を**疎結合**にする仕組み。クラスが他クラスを直接 `new` せず、コンテナ経由で受け取るため、**差し替え（本物↔モック）が容易**になる。
- Rails のように「グローバルに `User.find` を呼ぶ」のではなく、「必要な依存を明示的に宣言して注入される」スタイル。何に依存しているかがクラス先頭の `include Deps[...]` を見れば分かる。
- 自動登録のおかげで、ファイルを正しい場所に正しい名前で置くだけで、コンテナのキーとして使えるようになる（設定レスで配線される）。

## 基本の書き方（コード）
```ruby
# app/operations/books/create.rb
# → コンテナキーは "operations.books.create"（app/ 以下のパス由来）
module MyApp
  module Operations
    module Books
      class Create
        # 依存を注入。キー末尾がメソッド名になる
        # "repositories.book_repo" → book_repo メソッドが生える
        include Deps["repositories.book_repo"]

        def call(attrs)
          book_repo.create(attrs)
        end
      end
    end
  end
end
```

```ruby
# 別名で注入したい / ネストを潰したい場合はハッシュ指定
class SomeService
  include Deps[
    repo: "repositories.book_repo",   # repo メソッドとして使える
    "validation.book_contract"        # validation.book_contract → book_contract
  ]
end
```

```ruby
# 注入されたものはコンストラクタ引数なので、テストで差し替えられる
service = SomeService.new(repo: fake_repo, book_contract: fake_contract)
```

```ruby
# コンテナから直接取り出す（テストやrake等）
MyApp::App["repositories.book_repo"]   # 登録済みコンポーネントを取得
```

## 実務での使い方・定番パターン
- **キーはパスから機械的に決まる**：`app/repositories/book_repo.rb` → `"repositories.book_repo"`、`app/operations/books/create.rb` → `"operations.books.create"`。ディレクトリ区切りが `.` になる。
- **`include Deps["a.b.c"]` で末尾 `c` がインスタンスメソッド名**になる。衝突・読みやすさのため `include Deps[short: "a.b.c"]` のように**別名指定**するのが実務では多い。
- **依存の差し替え（テスト）**：`Deps` はコンストラクタ注入なので `Klass.new(dep: double)` でOK。あるいはコンテナ自体を `stub`／`register` で差し替える（`MyApp::App.stub("repositories.book_repo", fake)`）。→ [testing.md](./testing.md)
- **登録対象の調整**：すべてを自動登録したくない場合、`config.component_dirs` や `register` で明示登録・除外を制御できる。外部ライブラリ等は手動 `register` で持ち込む。
- **Slice ごとにコンテナが分かれる**：スライス内のキーはそのスライスのコンテナで解決される。スライスをまたぐ参照は別途公開設定が要る。→ [project_structure.md](./project_structure.md)
- **アクション・ビューも被注入側**：`include Deps[...]` はアクション／ビュー／任意のコンポーネントどこでも使える。アクションは依存を注入してロジックを委譲する薄い層に保つ。→ [actions.md](./actions.md)

## ハマりどころ / アンチパターン
- **キー（文字列）とファイルパスの不一致**：`Deps["repositories.book_repo"]` と書いたのに実体が `app/repos/book_repo.rb` だと解決できず起動エラー。**パス規約を厳守**する。これが最頻の詰まり。
- **メソッド名の衝突**：複数の依存が同じ末尾名（例 `repo`）になると上書き／混乱する。別名（ハッシュ指定）で解決する。
- **自動登録の対象外を当てにする**：`lib/` や設定で除外したディレクトリは登録されない。「置けば必ず使える」ではなく、コンポーネントディレクトリ配下かを確認する。
- **コンテナを `require` 直叩きで回避する**：手で `require` して `new` すると、DIの利点（差し替え・配線）が消える。`Deps` を通す。
- **循環依存**：A が B を、B が A を注入すると解決できない。責務を見直すか、片方を遅延参照にする。
- **読み込み順・boot の誤解**：コンポーネントは遅延解決される。起動時に全部が即 `new` されるわけではない、という前提でデバッグする。

## 関連
[project_structure.md](./project_structure.md) / [persistence_rom.md](./persistence_rom.md) / [actions.md](./actions.md) / [testing.md](./testing.md)
