# 始め方（Getting Started）（Sinatra）

## ひとことで言うと
Sinatraを動かすまでの最短手順。**インストール → アプリを書く → サーバで起動**の3ステップ。書き方には1ファイルで完結する **classic** と、クラスに包む **modular** の2スタイルがある。

## 役割・なぜ必要か
- Sinatraは「全部入り」ではないので、Railsの `rails new` のような雛形生成は無い。**自分で最小の構成を用意する**必要がある。
- 小さなアプリなら1ファイル（classic）で十分。複数アプリをマウントしたり、設定を整理したくなったら modular に移行する。
- どちらも実体は **Rack アプリ**。だから起動方法（`ruby app.rb` / `rackup` / Puma 直叩き）が共通の作法で語れる。

## 基本の書き方（コード）
```ruby
# インストール（単発）
#   gem install sinatra puma
#
# Gemfile（プロジェクトで管理する場合・推奨）
# source "https://rubygems.org"
# gem "sinatra", "~> 4.0"   # 3系なら "~> 3.0"
# gem "puma"                 # 本番想定の定番サーバ
# gem "rackup"               # rackup コマンド（Rack 3 で別gem化）
#   → bundle install

# --- classicスタイル: app.rb ---
require "sinatra"          # これだけでDSLがトップレベルに生える

get "/" do
  "Hello, Sinatra!"
end

get "/hello/:name" do
  "Hi, #{params[:name]}"
end
#   起動: ruby app.rb   → http://localhost:4567
```

```ruby
# --- modularスタイル: app.rb ---
require "sinatra/base"     # base だけ読む（トップレベルを汚さない）

class App < Sinatra::Base
  get "/" do
    "Hello from modular"
  end

  # 直接 ruby app.rb で起動したいとき用（任意）
  run! if app_file == $0
end
```

```ruby
# config.ru（rackup / Puma から起動する入口）
require "./app"
run App           # modular はクラスを run に渡す
# classic の場合は: require "./app"; run Sinatra::Application
```

## 実務での使い方・定番パターン
- **開発初期は classic** で素早く。ルートが増えて設定が散らかってきたら **modular へ移行**。
- 本番は **Puma + config.ru** が定番。`rackup` か `bundle exec puma config.ru` で起動する。
- 依存は必ず **Gemfile + Bundler** で固定（`bundle exec` 経由で実行）。バージョン差でDSLが微妙に変わるため。
- ポートやbindは `set :port, 4567` / `set :bind, "0.0.0.0"`（classic）や CLI `-p`、Pumaの設定で調整。
- 複数のSinatraアプリを1プロセスに同居させるなら modular にして `map "/admin" { run Admin }` のようにマウントする。

## ハマりどころ / アンチパターン
- **classic と modular の混同**：`require "sinatra"` は読み込むだけでルートが「起動する」性質を持つ（at_exitで自動run）。modularでこれを混ぜると二重起動・意図しない挙動になる。modularでは必ず `require "sinatra/base"`。
- **「Sinatraは特別なサーバ」だと思う誤解**：実体はRackアプリ。`config.ru` を書けば Puma/Unicorn/任意のRackサーバに載る。サーバ固有の話はRack側の知識で解決する。
- **`run! if app_file == $0` を書き忘れる**：modularを `ruby app.rb` で直接起動できず「何も起きない」。config.ru経由なら不要。
- **Rack 3系で `rackup` が無い**と言われる：`rackup` gem を別途入れる（Rack 3 で本体から分離）。
- グローバルに `require "sinatra"` したライブラリを混ぜると、テスト時に勝手にサーバが立つ事故。ライブラリ側は base を使う。

## フォルダ構成（始動直後）
```
myapp/
├── app.rb                       # ルーティングも処理もここに（軽量スタート）
├── app/                         # 規模が育ったら分割（# 自分で作る）
│   ├── routes/                  # ルート定義を機能別に分割（# 自分で作る）
│   ├── models/                  # データモデル（# 自分で作る）
│   └── helpers/                 # 共通ヘルパー（# 自分で作る）
├── config.ru                    # rackup / Puma 起動の入口
├── Gemfile                      # 依存gem定義
├── Gemfile.lock                 # 依存の固定版
├── Rakefile                     # rakeタスクのエントリ（# 自分で作る）
├── views/                       # テンプレート
│   ├── layout.erb               # 共通レイアウト
│   └── index.erb                # 各画面テンプレート
├── public/                      # 静的ファイル
│   ├── css/                     # スタイルシート
│   └── js/                      # クライアントJS
├── spec/                        # テスト（# 自分で作る）
│   └── spec_helper.rb           # テスト共通設定
└── .env                         # 環境変数（gitignore対象・# 自分で作る）
```
- 公式の雛形は無い。1ファイル（app.rb）から始め、規模に応じて app/ 配下に分割していく。

## 関連
[routing.md](./routing.md) / [rack_and_filters.md](./rack_and_filters.md)
