# Rails 6 実務リファレンス（索引）

> **この版 = Rails 6 系（6.0〜6.1）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。
> （周辺インフラは版共通の [`../周辺インフラ/`](../周辺インフラ/)。Rails 7 版は [`../rails7/`](../rails7/)。）

## この版のポイント（Rails 6 で何が標準か）
- **Webpacker が標準** … アプリJSは `app/javascript/packs/application.js`、依存は `yarn add`。CSS/画像は Sprockets が併存。
- **Zeitwerk オートローダーが新既定** … 定数名とファイル名・ディレクトリ名の厳密一致が必須（classic も選択可）。
- **複数DB対応** … `database.yml` の primary/replica、`connects_to`/`connected_to`、GET/HEAD を replica に自動振り分け。
- **Action Text**（Trix・`has_rich_text`）／ **Action Mailbox**（受信メール）／ **Active Storage**（variant）。
- **並列テスト**（`parallelize`）、`insert_all`/`upsert_all`、`pick`。

## 7 との違い（Rails 6 でやってはいけない/違う点）
- フロントは **Turbolinks 5 + rails-ujs**。**Hotwire（Turbo/Stimulus）は標準でない**（turbo-rails を 6.1 に後付けは可能だが既定でない）。
- 削除リンクは `link_to "削除", path, method: :delete, data: { confirm: "..." }`（rails-ujs）。`turbo_method` は使わない。
- 作成・更新の失敗時は単に `render :new` でよい。**`status: :unprocessable_entity` は不要**（Turbo前提でないため）。
- JSライブラリ追加は `yarn add`（Webpacker）。**importmap ではない**。
- イベント初期化は `DOMContentLoaded` ではなく **`turbolinks:load`**。

## 6.0 → 6.1 の追加
- **`strict_loading`**（N+1ガード。`strict_loading` / `config.active_record.strict_loading_by_default`）。
- **`where.missing`**（関連が無いレコードを引く）。
- **`delegated_types`**（ポリモーフィックの代替設計）。
- **水平シャーディング**（`connected_to(shard: :shard_one)`）。
- `form_with` の **`local` が既定**に変更（6.0 は既定 remote=ajax）。

## リクエストの流れ（全体像）
```
ブラウザ → ルーティング(routes.rb) → コントローラ#アクション
        → モデル(Active Record)でDB操作 → ビュー(View)でHTML生成 → レスポンス
（Turbolinks 5 でページ遷移を高速化 / API mode なら JSON）
```

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（`rails new`〜Webpacker〜起動〜scaffold）

### コア（MVC）
- [request_flow.md](./request_flow.md) … リクエスト/処理の流れ・各層は何を返すか（全体像）
- [routing.md](./routing.md) … ルーティング
- [controller.md](./controller.md) … コントローラ
- [model.md](./model.md) … モデル
- [view.md](./view.md) … ビュー
- [helper.md](./helper.md) … ヘルパー
- [active_record.md](./active_record.md) … DB / Active Record（マイグレーション・関連・クエリ・N+1・複数DB・strict_loading・insert_all）

### フロント（Rails 6 特有）
- [webpacker.md](./webpacker.md) … Webpacker（標準のJSビルド）
- [assets.md](./assets.md) … アセット（Sprockets＋Webpacker の役割分担）
- [javascript.md](./javascript.md) … RailsでのJSの書き方（rails-ujs + Turbolinks 5 + Webpacker。タスク別の具体コード）

### 関連事項（周辺インフラ）→ 版共通の [`../周辺インフラ/`](../周辺インフラ/) に集約
- [redis](../周辺インフラ/redis.md) … キャッシュ / ジョブの土台
- [sidekiq](../周辺インフラ/sidekiq.md) … Redis前提の定番ジョブ基盤
- [unicorn](../周辺インフラ/unicorn.md) … アプリケーションサーバ
- （DBエンジン・Puma・S3 等も `../周辺インフラ/` に順次追加）

### リクエスト処理・設定
- [strong_parameters.md](./strong_parameters.md) … Strong Parameters
- [filters.md](./filters.md) … before_action などフィルタ
- [partial_layout.md](./partial_layout.md) … partial / layout
- [session_cookie_flash.md](./session_cookie_flash.md) … session / cookie / flash
- [config_credentials.md](./config_credentials.md) … 設定 / credentials / ENV（環境別credentials）

### 認証・非同期・メール
- [auth.md](./auth.md) … 認証・認可（has_secure_password / devise / Pundit）
- [active_job.md](./active_job.md) … Active Job（バックグラウンドジョブ）
- [action_mailer.md](./action_mailer.md) … Action Mailer（送信メール）
- [action_mailbox.md](./action_mailbox.md) … Action Mailbox（受信メール。Rails 6 特有）

### 設計・整理パターン
- [concern.md](./concern.md) … Concern
- [module.md](./module.md) … module（名前空間 / ミックスイン / Engine）
- [service_form.md](./service_form.md) … Service / Form オブジェクト

### Rails 6 特有の機能
- [zeitwerk.md](./zeitwerk.md) … Zeitwerk オートローダ（よくある NameError）
- [multiple_databases.md](./multiple_databases.md) … 複数DB（primary/replica・connected_to・シャーディング）
- [action_text.md](./action_text.md) … Action Text（リッチテキスト / Trix）
- [active_storage.md](./active_storage.md) … Active Storage（ファイル添付 / variant）

### テスト・安全・運用
- [testing.md](./testing.md) … テスト（Minitest / RSpec / 並列テスト / system test）
- [security.md](./security.md) … セキュリティ（CSRF / XSS / SQLi / Strong Params）
- [console.md](./console.md) … rails console / コマンド
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（Zeitwerk・Webpacker・複数DB の早見表）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Rails 6）

## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連
[xxx.md](./xxx.md) / ...
```
