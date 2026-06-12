# はじめに / 始め方（Hanami 2）

## ひとことで言うと
Hanami 2 アプリの**作り方・動かし方の入口**。`hanami new` で雛形を生成し、`hanami server` で起動、`hanami console` で対話する。Hanami 2 は v1 から全面刷新されており、**Rails とは別物の前提**で触れる。

## 役割・なぜ必要か
- Hanami 2 は「**DIコンテナ＋明示的なコンポーネント分割**」が核。Rails の「規約で勝手に繋がる」感覚ではなく、`app/` 配下のクラスがコンテナに登録され、必要なものを `Deps[...]` で注入して使う。
- まず最小構成を生成して `server` / `console` を触ることで、「アクション＝1クラス」「ビューは独立」「依存は注入」という Hanami 流の世界観を体で掴むのが目的。
- Rails ほど generator に頼らない（控えめ）。ファイルは**自分で置く**ことが多く、その代わり構成が読みやすい。

## 基本の書き方（コード）
```bash
# 1) gem をインストール（Ruby 3.x 前提）
gem install hanami

# 2) 新規アプリ生成（bookshelf という名前で）
hanami new bookshelf
cd bookshelf

# 3) 依存を入れる（生成された Gemfile を使う）
bundle install

# 生成される主なディレクトリ（2.1+ は app/ 中心）
# .
# ├── app/            # メインアプリ（アクション/ビュー/オペレーション等）
# │   ├── actions/
# │   ├── views/
# │   └── templates/
# ├── config/         # routes.rb / app.rb / settings.rb など
# ├── lib/            # ドメインコード・共有ライブラリ
# ├── slices/         # サブアプリ（必要になったら）
# ├── spec/           # RSpec
# └── Gemfile
```

```bash
# 4) サーバ起動（既定は http://localhost:2300）
hanami server
# 開発中は Puma が立ち、コード変更は自動リロードされる

# 5) コンソール（アプリのコンテナを読み込んだ状態で対話）
hanami console
```

```ruby
# console の中ではアプリのコンテナにアクセスできる
# 例: コンテナからコンポーネントを取り出す
Hanami.app["settings"]          # 設定オブジェクト
Hanami.app.keys                 # 登録済みキー一覧を確認
Hanami.app["my_app.some_service"]  # 任意のコンポーネントを解決
```

## 実務での使い方・定番パターン
- **最初に `config/routes.rb` と `app/` を見る**。ルートが `to:` でアクションキー（例 `"home.show"`）を指し、それが `app/actions/home/show.rb` の `Home::Show` に対応する、という対応関係を掴むのが第一歩。→ [routing.md](./routing.md)
- 機能追加は「**アクション → ビュー → テンプレート → （必要なら）永続化**」の順に小さなクラスを置いていく。1責務1クラスが基本。
- `hanami console` は実装確認の主戦場。`Hanami.app.keys` で何が登録されているか確認しながら進めると、コンテナの全体像が掴める。→ [dependency_injection.md](./dependency_injection.md)
- アプリが育って機能群が独立してきたら **slices** に切り出す。最初から slices を作らず、`app/` で始めて必要時に分割するのが定石。→ [slices.md](./slices.md)
- オートロードは **Zeitwerk**。ファイル名とクラス名・モジュールパスが一致している必要がある（`app/actions/books/index.rb` → `Books::Index`）。

## ハマりどころ / アンチパターン
- **Rails 感覚でファイルを探さない**。`app/models/` も `ActiveRecord` も無い。モデル相当は ROM の Relation / Repository、ロジックは `lib/` や operations に置く。「あるはずの規約」を前提にすると迷子になる。
- **generator に期待しすぎない**。Hanami 2 は scaffold が控えめ。`hanami generate` で出せるものは限られ、多くは手で置く。これは「魔法を減らして見通しを良くする」設計思想によるもの。
- **Hanami 1.x の情報を混ぜない**。ネット上には v1 系の記事が多く残るが、ディレクトリ構成・DSL・概念が全面的に違う。必ず **2.x（できれば 2.1 以降）** の前提か確認する。
- **Zeitwerk の命名ズレ**でロード失敗。`NameError`/`uninitialized constant` が出たら、まずファイルパスと定数名（モジュールのネスト）が一致しているか疑う。
- `hanami server` が起動しない時は、`bundle install` 漏れ・Ruby バージョン不一致・`config/` の設定不足を順に確認する。設定は起動時に検証される（足りないと早期に落ちる）。→ [settings_config.md](./settings_config.md)

## フォルダ構成（始動直後）
```
myapp/
├── app/                                  # アプリ本体
│   ├── action.rb                         # 基底アクション【生成】
│   ├── view.rb                           # 基底ビュー【生成】
│   ├── actions/                          # 1アクション=1クラス
│   │   └── users/
│   │       ├── index.rb                  # 自分で作る
│   │       └── show.rb
│   ├── views/
│   │   └── users/
│   │       └── index.rb                  # ビューオブジェクト（自分）
│   ├── templates/                        # テンプレート（.html.erb）
│   │   ├── layouts/app.html.erb          # 共通レイアウト【生成】
│   │   └── users/index.html.erb          # 自分で作る
│   ├── assets/                           # JS/CSS（esbuild）
│   │   └── js/app.js  css/app.css        # 【生成】
│   ├── relations/                        # ── データ層(ROM) ── DBテーブル＝リレーション
│   │   └── users.rb                      # users テーブル（自分で作る）
│   ├── repos/                            # リポジトリ＝取得・保存の窓口
│   │   └── user_repo.rb                  # Hanami::DB::Repo（自分で作る）
│   └── structs/                          # 構造体＝取り出した1件（entity相当・不変）
│       └── user.rb                       # Hanami::DB::Struct（自分で作る）
├── config/
│   ├── app.rb                            # アプリ設定【生成】
│   ├── routes.rb                         # ルーティング【生成】
│   ├── settings.rb                       # 型付き設定（ENV）【生成】
│   ├── puma.rb
│   └── db/                               # DB（Hanami 2.2〜）
│       ├── migrate/                      # マイグレーション（自分で作る）
│       │   └── 20250101_create_users.rb
│       └── structure.sql                 # スキーマ（自動生成）
├── lib/
│   └── myapp/                            # ドメインロジック（Railsと違い分離）
│       └── types.rb                      # 【生成】
├── slices/                               # 機能スライス（モジュール分割・自分で作る）
│   └── admin/
├── spec/
│   ├── spec_helper.rb                    # 【生成】
│   └── requests/
├── config.ru                             # Rack起動【生成】
├── Gemfile  Gemfile.lock                 # 依存【生成】
├── Rakefile                              # 【生成】
└── .env                                  # 環境変数【生成】
```
- Hanami 2 は **1アクション=1クラス**（`actions/users/index.rb`）。Railsの「1コントローラに複数アクション」と違う。
- **データ層は ActiveRecord と別物（ROMベース）**：Railsの「model」1つではなく、**Relations（テーブル定義）/ Repos（取得・保存の窓口）/ Structs（取り出した1件＝entity相当）** の3つに分かれる。マイグレーションは `config/db/migrate/`、`hanami db migrate` で適用。※DB層は **Hanami 2.2〜** で標準化。
- **ビジネスロジックは `lib/` に分離**、機能は **`slices/`** でモジュール化（軽量な分割アプリ）。
- **【生成】= `hanami new`。** `actions/`・`views/`・`templates/`・`relations/`・`repos/`・`structs/` の中身は自分で作る。

## 関連
[project_structure.md](./project_structure.md) / [routing.md](./routing.md) / [dependency_injection.md](./dependency_injection.md)
