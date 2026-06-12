# 設定（Settings / Config）（Hanami 2）

## ひとことで言うと
アプリ全体の設定値を **`config/settings.rb`** に **`Hanami::Settings`** クラスとして定義し、ENV から **型付き**（dry-types）で読み込む仕組み。値は DI コンテナに `"settings"` として登録され、どこからでも安全に参照できる。

## 役割・なぜ必要か
- 環境変数（ENV）は**ただの文字列**。`"3"`、`"true"`、`"1.5"` をそのまま使うとバグの元。**型変換と必須チェックを起動時に1か所で**やるのが目的。
- 必須の設定（DB URL や API キー）が欠けていれば、**起動時に即落とす**（fail fast）。本番で初めて気づく事故を防ぐ。
- 設定の参照経路が `Hanami.app["settings"]` / `Deps["settings"]` に一本化され、散らばった `ENV[...]` 直読みを撲滅できる。

## 基本の書き方（コード）
```ruby
# config/settings.rb
require "hanami/settings"
require "dry/types"

module MyApp
  class Settings < Hanami::Settings
    Types = Dry.Types()

    # 必須（欠けると起動時エラー）。constructor で型変換
    setting :database_url, constructor: Types::String

    # 真偽値（"true"/"false"/"1"/"0" を Bool に）
    setting :redis_enabled, default: false, constructor: Types::Params::Bool

    # 整数（"3" → 3）。範囲制約も付けられる
    setting :max_retries, default: 3,
            constructor: Types::Params::Integer.constrained(gteq: 0)

    # 任意（無ければ nil）
    setting :sentry_dsn, constructor: Types::String.optional

    # 環境ごとに既定を変えたい場合は Hanami.env を見て分岐
    setting :session_secret, constructor: Types::String
  end
end
```
```bash
# .env（dotenv 経由で読み込まれる。本番は実 ENV を使う）
DATABASE_URL=postgres://localhost/myapp_dev
REDIS_ENABLED=true
MAX_RETRIES=5
SESSION_SECRET=devsecret
```
```ruby
# 参照1：コンテナから直接取得
settings = Hanami.app["settings"]
settings.database_url   # => "postgres://..."（String 型保証）
settings.max_retries    # => 5（Integer）

# 参照2：Deps 経由で依存注入（推奨）
class CreateUser
  include Deps["settings"]

  def call(attrs)
    raise "リトライ上限超過" if attrs[:retries] > settings.max_retries
    # ...
  end
end
```

## 実務での使い方・定番パターン
- **必須は constructor のみ・任意は default 付き**：default を付けない設定は ENV 欠落で起動時に落ちる。「無いと動かない値」は意図的に default を付けないのが鉄則。
- **型は `Types::Params::*` を使う**：ENV は文字列なので `Params::Bool` / `Params::Integer` を使う（`Coercible` 系でも可）。素の `Types::Integer` だと文字列 `"3"` を弾く。
- **秘密情報は ENV / シークレットマネージャ経由**：`config/settings.rb` に値そのものを書かない。定義（型・必須）だけを書く。
- **環境別の差分**は `Hanami.env`（`:development` / `:test` / `:production`）で分岐するか、環境ごとの `.env` ファイルを使う。
- **テストでは settings をスタブ**：`Hanami.app.container.stub("settings", fake_settings)` で差し替え可（→ [testing.md](./testing.md)）。

## ハマりどころ / アンチパターン
- **型を指定し忘れる**：constructor 無しだと文字列のまま。`max_retries > 3` のような数値比較が `"5" > 3` で例外、あるいは意図しない挙動に。**必ず型を付ける**。
- **`Types::Integer` と `Types::Params::Integer` の取り違え**：前者は `"3"` を弾く、後者は変換する。ENV 由来は必ず `Params`（または `Coercible`）系。
- **必須 ENV の欠落に気づかない**：default を付けてしまうと、本番で値が無くても既定値で「動いてしまい」事故が遅延発覚。**必須なら default を付けない**。
- **settings の参照方法を間違える**：アプリ内で `ENV["..."]` を直読みすると型・必須チェックを素通りする。常に `Deps["settings"]` か `Hanami.app["settings"]` 経由にする。
- **`.env` をコミット**：実値入りの `.env` は `.gitignore`。リポジトリには `.env.example`（キーのみ）を置く。
- **slice 固有設定をアプリ全体に混ぜる**：スコープが曖昧になる。slice 境界を意識する（→ [slices.md](./slices.md)）。

## 関連
[dependency_injection.md](./dependency_injection.md) / [testing.md](./testing.md) / [slices.md](./slices.md)
