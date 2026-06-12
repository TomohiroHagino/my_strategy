# Zeitwerk（Rails 6）

## ひとことで言うと
Rails 6 の既定オートローダーで、定数名とファイル名・ディレクトリ名の厳密一致を前提に `require` なしで定数を解決する。命名がズレると `NameError`（または `LoadError`）になる。

## 役割・なぜ必要か
- Rails 6 では `app/` 配下や `eager_load_paths` のクラス・モジュールを、参照された瞬間（lazy）または起動時（eager）に自動ロードする。
- Rails 5 までの `classic` オートローダーは「定数が無ければ推測してファイルを探す」方式だったが、Zeitwerk は「ファイルパスから定数名を逆算して照合する」方式に変わった。このため命名規約の逸脱が起動時・チェック時に検出できる。
- 既定が Zeitwerk なので、`require` を手書きする必要がなく、開発環境では編集後の自動リロードが効く。

## 基本の書き方（コード）

### 命名規約（パス → 定数の対応）
```
app/models/user.rb                 -> User
app/models/api_client.rb           -> ApiClient
app/services/payment/processor.rb  -> Payment::Processor
app/controllers/admin/users_controller.rb -> Admin::UsersController
app/models/concerns/trackable.rb   -> Trackable
```
- `payment/processor.rb` のように **ディレクトリがそのまま namespace** になる。`Payment::Processor` を定義するファイルは必ず `payment/processor.rb`。
- ファイル名は `snake_case`、定数は対応する `CamelCase`。1ファイル1トップレベル定数が原則。

### 頭字語（acronym）の設定
```ruby
# config/initializers/inflections.rb
ActiveSupport::Inflector.inflections(:en) do |inflect|
  inflect.acronym "API"
  inflect.acronym "HTML"
end
```
- 設定後の対応：
```
app/controllers/api_controller.rb  -> APIController（acronym設定時）
app/lib/html_parser.rb             -> HTMLParser
```
- acronym を設定しないと `api_controller.rb` は `ApiController` を期待する。`APIController` と書くとズレてエラーになる。

### autoloader 設定とチェック
```ruby
# config/application.rb
# config.autoloader = :zeitwerk  # Rails 6 の既定（明示不要）
# config.autoloader = :classic   # 旧方式に戻す場合のみ
```
```bash
# 命名規約の不一致を起動せずに一括検査する
bin/rails zeitwerk:check
```
```ruby
# コンソールでローダーを直接確認
Rails.autoloaders.main          # アプリ本体のローダー
Rails.autoloaders.main.dirs     # autoload 対象ディレクトリ一覧
```

### eager load と lazy load
- 開発環境（`config.eager_load = false`）：参照時に都度ロード（lazy）。編集後にリロードされる。
- 本番環境（`config.eager_load = true`）：起動時に全定数をロード（eager）。命名ズレは起動時に即クラッシュするため、デプロイ前に `zeitwerk:check` で検出する。

## 実務での使い方・定番パターン
- 命名規約を最初から厳守する。`User` を定義するなら `user.rb`、`Payment::Processor` なら `payment/processor.rb` に置く。途中で破ると本番起動時に落ちる。
- 頭字語を含むクラス（`API`, `URL`, `HTTP`, `CSV`, `SSO` 等）は `inflections.rb` に `acronym` を登録してから命名を決める。
- `app/services`, `app/forms` 等の独自ディレクトリは Rails 6 では既定で autoload 対象（`app/*` は自動で含まれる）。namespace を切る場合はサブディレクトリを作る。
- `require` を書かない。`app/` 配下の他クラスは定数参照だけで解決される。`require` するとリロード機構と二重定義で壊れる。
- CI に `bin/rails zeitwerk:check` を組み込み、命名ズレをデプロイ前に検出する。
- classic から移行する場合：`config.autoloader = :zeitwerk` にした上で `zeitwerk:check` を通し、報告された全ファイルの命名を直してから移行を確定する。

## ハマりどころ / アンチパターン

### NameError: expected file ... to define constant
```
NameError: expected file /app/models/api_client.rb to define constant ApiClient,
but didn't (Zeitwerk::NameError)
```
- 原因：`api_client.rb` の中で `class APIClient` と書いている（acronym設定済み）、あるいは acronym 未設定なのに `class ApiClient` を期待されていてファイル側が `APIClient` になっている、などパスと定数のズレ。
- 対処：ファイルパスから期待される定数名を確認し、`class`/`module` 名をそれに一致させる。頭字語が絡むなら `inflections.rb` に `acronym` を追加してから命名を統一する。

### ファイル名と定数名のズレ
```
Zeitwerk::NameError: expected file .../user_account.rb to define constant UserAccount
```
- 原因：`user_account.rb` に `class Account` 等、別名を定義している。
- 対処：ファイル名と定数名を 1 対 1 に合わせる。`Account` を使いたいなら `account.rb` にする。

### 頭字語 inflection 未設定
- 症状：`APIController` を `api_controller.rb` に置くと「expected ... to define constant ApiController」。
- 対処：`inflect.acronym "API"` を設定する。設定しないなら定数名を `ApiController` にしてファイルも `api_controller.rb` のまま揃える。

### require してしまいリロードが壊れる
- 症状：開発環境でクラスを編集しても反映されない、または `superclass mismatch`/二重定義エラー。
- 原因：`app/` 配下のクラスを `require` で読み込んでいる。Zeitwerk が管理する定数を `require` すると、リロード時に古い定義が残り衝突する。
- 対処：`require` を削除し、定数参照だけにする。外部 gem は `require` してよいが、autoload 対象の自前クラスはしない。

### lib/ を autoload 対象にする設定
- 症状：`lib/` に置いた純粋な Ruby ユーティリティ（namespace無し・命名規約外）が `NameError` を起こす。
- 原因：`config.autoload_paths << Rails.root.join("lib")` 等で `lib/` を autoload に入れると、`lib/` 内の全ファイルが Zeitwerk の命名規約に従う必要が出る。
- 対処：規約に合わない `lib/` 配下は autoload に入れず、必要箇所で明示 `require` する。autoload させたいなら `lib/` 内も命名規約に従わせる。

### concerns の命名
- 症状：`app/models/concerns/trackable.rb` が「expected to define Trackable」で落ちる。
- 原因：`concerns` ディレクトリは namespace に含まれない（特別扱い）。`Concerns::Trackable` ではなく `Trackable` を定義する必要がある。
- 対処：concern は `module Trackable` のようにトップレベル名で定義する。

## 関連
[getting_started.md](./getting_started.md) / [config_credentials.md](./config_credentials.md) / [concern.md](./concern.md) / [assets.md](./assets.md) / [pitfalls.md](./pitfalls.md)
