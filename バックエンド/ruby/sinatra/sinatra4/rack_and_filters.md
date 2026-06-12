# Rack・ミドルウェア・フィルタ（Rack & Filters）（Sinatra 3/4）

## ひとことで言うと
**SinatraはRackアプリ**である、という土台の話。`use SomeMiddleware` でRackミドルウェアを挟み、`before`/`after` で全ルート（やパターン）の前後処理を差し込み、`helpers do ... end` で共通メソッドを足し、`set` で設定する。これらが「ルート以外の組み立て」の道具。

## 役割・なぜ必要か
- Sinatraは**Rackという共通インターフェースの上の薄い層**。だからログ・認証・セッション・圧縮などは**Rackミドルウェアをそのまま積める**（Sinatra専用の作法は要らない）。
- `before`/`after` フィルタは「毎リクエスト共通でやりたいこと」（認証チェック・計測・共通ヘッダ）の置き場。各ルートに同じ前処理をコピペしないため。
- `helpers` は**テンプレ/ルートから呼べる共通メソッド**（整形・ログイン判定など）の置き場。`set` はアプリ全体の設定スイッチ。

## 基本の書き方（コード）
```ruby
require "sinatra"
require "rack/protection"

# ── ミドルウェア（Rackの作法そのまま）──
use Rack::Protection            # CSRF等の保護を全体に
use Rack::Deflater              # レスポンス圧縮
# ※ classic スタイルでは Rack::Session などは下記 set で有効化される

# ── 設定（set）──
set :sessions, true             # セッション有効化
set :show_exceptions, false     # 本番では例外画面を出さない

# ── helpers（ルート/テンプレから呼べる共通メソッド）──
helpers do
  def logged_in?   = session[:user_id] ? true : false
  def current_user = session[:user_id]
end

# ── before：全ルートの前に毎回走る ──
before do
  content_type :json if request.path.start_with?("/api")
end

# ── before（パターン指定）：/admin/* だけ ──
before "/admin/*" do
  halt 401, "auth required" unless logged_in?
end

# ── after：全ルートの後に走る（後処理・共通ヘッダ等）──
after do
  response.headers["X-App"] = "sinatra"
end

get("/")          { "hello" }
get("/admin/top") { "secret" }
```

```ruby
# modular スタイルでも同じ。クラス内に use/before/helpers を書く
require "sinatra/base"

class App < Sinatra::Base
  use Rack::Protection::AuthenticityToken   # 個別の保護を選んで積む
  enable :sessions

  before { @started = Time.now }
  after  { logger.info("took #{Time.now - @started}s") }

  helpers do
    def h(text) = Rack::Utils.escape_html(text)   # テンプレ用の整形など
  end

  get("/") { "ok" }
end
```

## 実務での使い方・定番パターン
- **`use` でミドルウェアを積む**：認証・ログ・圧縮・レート制限などはRackミドルウェアで横断的に。Rackエコシステムの資産がそのまま使える。
- **`Rack::Protection` は一部が既定で有効**：Sinatraはセッション有効時などに保護の一部を既定で入れる。CSRFトークン（`Rack::Protection::AuthenticityToken`）は用途に応じて明示的に積む/確認する。
- **`before` は前処理、`after` は後処理**。認証ガード・共通ヘッダ・計測の定番置き場。パターン指定（`before "/admin/*"`）で対象を絞れる。
- **`helpers` に共通メソッド**を集約（整形・ログイン判定）。テンプレからも呼べる。→ [views.md](./views.md)
- **`set`/`enable`/`disable`** で設定スイッチ（`set :sessions, true` ≒ `enable :sessions`）。環境別の出し分けは environments と組む。→ [config_testing.md](./config_testing.md)

## ハマりどころ / アンチパターン
- **ミドルウェアの順序が結果を変える（最重要）**：`use` は**書いた順**に外側→内側へ積まれる。認証・セッション・圧縮の前後関係を間違えると「セッションが無い状態で認証が走る」等が起きる。順序は意図して並べる。
- **`before` は“全ルート”に効く**：パターン無しの `before` は **すべてのリクエスト**（静的ファイル含むことも）に走る。重い処理や副作用を入れると全体が遅くなる/壊れる。対象を絞るなら `before "/path/*"`。
- **`halt` の使い所**：フィルタ内で `halt` すると以降のルート処理を止められる（認証ガードの定番）が、`after` 側の後処理が走るか/フィルタの実行順を把握していないと、ヘッダ付与漏れ等が起きる。
- **CSRF保護の取り違え**：`Rack::Protection` を入れた“つもり”でトークン系（`AuthenticityToken`）が効いておらず、POSTで弾かれる/逆に無防備、が起きやすい。何が有効か明示的に確認する。
- **helpersに業務ロジックを溜めすぎ**：肥大化したらservice相当の別オブジェクトへ。テンプレ整形以外は持ち込みすぎない。
- modular で `use` の置き場（クラス内）を間違えると効かない。classic（トップレベル）との書き分けに注意。→ [config_testing.md](./config_testing.md)

## 関連
[config_testing.md](./config_testing.md) / [views.md](./views.md) / [request_response.md](./request_response.md)
