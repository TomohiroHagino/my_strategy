# Rails 4 実務リファレンス（索引）

> **この版 = Rails 4 系（4.0〜4.2）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。
> （周辺ミドルウェアは版共通の `../周辺インフラ/` に集約。重複生成しない。）

## この版のポイント（Rails 4 で何が標準か）
- **Strong Parameters が標準**。`attr_accessible`/`attr_protected`（Rails 3の手法）は本体から外れ gem 化された。許可キーはコントローラ側で `params.require(...).permit(...)`。
- **フロント/JS は jQuery 系**：`jquery_ujs`（jQueryベースUJS＝`data-remote`/`data-method`/`data-confirm`）＋ **Turbolinks Classic（3系）** ＋ **`*.js.erb`（SJR）**。Hotwire / Turbo / Stimulus は無い。
- **アセットは Sprockets のみ**（`app/assets`、CoffeeScript / Sass）。Webpacker は無い。
- 設定の秘密情報は **`config/secrets.yml`（4.1〜）**。credentials は無い（7系の `credentials.yml.enc` に相当する仕組みは未導入）。
- **Active Job は 4.2 で導入**。それ以前（4.0/4.1）は Active Job 自体が無く、Sidekiq/Resque/DelayedJob を直接叩く。Action Cable / Active Storage も無い（アップロードは paperclip / carrierwave）。
- モデルは **`ActiveRecord::Base` を直接継承**（`ApplicationRecord` は無い＝Rails 5から）。`belongs_to` は**既定で必須ではない**（presence チェックは付かない）。`enum` は 4.1〜。
- タスクランナーは **`rake`**（`rake db:migrate`）。`bin/rails` も使えるが `rake` が主流。`respond_to` / `respond_with` を多用。system test は無い。Spring は 4.1〜。

## リクエストの流れ（全体像）
```
ブラウザ → ルーティング(routes.rb) → コントローラ#アクション
        → モデル(Active Record)でDB操作 → ビュー(View)でHTML生成 → レスポンス
（Ajaxなら jquery_ujs + *.js.erb で部分更新 / API なら respond_with で JSON）
```

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（`rails new`〜起動〜scaffold）

### コア（MVC）
- [request_flow.md](./request_flow.md) … リクエスト/処理の流れ・各層は何を返すか（全体像）
- [routing.md](./routing.md) … ルーティング
- [controller.md](./controller.md) … コントローラ
- [model.md](./model.md) … モデル
- [view.md](./view.md) … ビュー
- [helper.md](./helper.md) … ヘルパー
- [active_record.md](./active_record.md) … DB / Active Record（マイグレーション・関連・クエリ・N+1）

### フロント
- [assets.md](./assets.md) … アセット（Sprockets / CoffeeScript / Sass）
- [javascript.md](./javascript.md) … RailsでのJSの書き方（jquery_ujs / Turbolinks Classic / `*.js.erb`）

### 関連事項（周辺インフラ）→ 版共通の [`../周辺インフラ/`](../周辺インフラ/) に集約
- [redis](../周辺インフラ/redis.md) … キャッシュ / ジョブの土台
- [sidekiq](../周辺インフラ/sidekiq.md) … Redis前提の定番ジョブ基盤（Rails 4 ではActive Jobを介さず直接、または4.2でActive Job経由）
- [unicorn](../周辺インフラ/unicorn.md) … Rails 4 時代の定番アプリサーバ（Pumaが既定になるのは5系）

### リクエスト処理・設定
- [strong_parameters.md](./strong_parameters.md) … Strong Parameters
- [filters.md](./filters.md) … `before_filter` → `before_action`（過渡期）
- [partial_layout.md](./partial_layout.md) … partial / layout
- [session_cookie_flash.md](./session_cookie_flash.md) … session / cookie / flash
- [config_secrets.md](./config_secrets.md) … 設定 / `secrets.yml` / ENV

### 認証・非同期・メール
- [auth.md](./auth.md) … 認証・認可（has_secure_password / devise / CanCanCan）
- [active_job.md](./active_job.md) … Active Job（**4.2で導入**。それ以前は無い）
- [action_mailer.md](./action_mailer.md) … Action Mailer

### 設計・整理パターン
- [concern.md](./concern.md) … Concern
- [module.md](./module.md) … module（名前空間 / ミックスイン / Engine）
- [service_form.md](./service_form.md) … Service / Form オブジェクト

### テスト・安全・運用
- [testing.md](./testing.md) … テスト（Minitest / RSpec。system test は無い）
- [security.md](./security.md) … セキュリティ（CSRF / XSS / SQLi / mass assignment）
- [console.md](./console.md) … `rails console` / `rake` コマンド
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（早見表）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Rails 4）

## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連
[xxx.md](./xxx.md) / ...
```
