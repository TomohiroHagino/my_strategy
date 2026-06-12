# Rails 7 実務リファレンス（索引）

> **この版 = Rails 7 系（7.0〜7.2）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。
> （他バージョンは一旦作らず Rails 7 のみ。周辺インフラは版共通の `../周辺インフラ/`。）

## この版のポイント（Rails 7 で何が変わったか）
- **Hotwire（Turbo / Stimulus）が標準** … JSを最小化したSPA風UIが既定路線。
- **importmap が既定**（Node不要でJSを使える）。jsbundling / cssbundling も選択可。Webpacker は卒業。
- **`encrypts`（暗号化属性）** が標準化。**非同期クエリ `load_async`** 導入。
- アセット配信に **Propshaft** を選べる（Sprockets脱却の流れ）。
- 7.1: `Dockerfile` 生成が既定・複合主キー対応。7.2: dev用Dockerfile / `bin/dev` 整備など。

## リクエストの流れ（全体像）
```
ブラウザ → ルーティング(routes.rb) → コントローラ#アクション
        → モデル(Active Record)でDB操作 → ビュー(View)でHTML生成 → レスポンス
（Hotwireなら部分更新 / API modeならJSON）
```

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（`rails new`〜起動〜scaffold）

### コア（MVC）
- [request_flow.md](./request_flow.md) … リクエスト/処理の流れ・各層は何を返すか（全体像）
- [routing.md](./routing.md) … ルーティングとは
- [controller.md](./controller.md) … コントローラとは
- [model.md](./model.md) … モデルとは
- [view.md](./view.md) … ビューとは
- [helper.md](./helper.md) … ヘルパーとは
- [active_record.md](./active_record.md) … DB / Active Record とは（マイグレーション・関連・クエリ・N+1）

### フロント
- [hotwire.md](./hotwire.md) … Hotwire（Turbo / Stimulus）とは
- [javascript.md](./javascript.md) … RailsでのJSの書き方（タスク別「過去→今」の具体例）
- [turbo_drive.md](./turbo_drive.md) … ページ遷移の高速化（Turbolinks→Turbo Drive の詳細）

### 関連事項（周辺インフラ）→ 版共通の [`../周辺インフラ/`](../周辺インフラ/) に集約
- [redis](../周辺インフラ/redis.md) … キャッシュ / ジョブ / Action Cable の土台
- [sidekiq](../周辺インフラ/sidekiq.md) … Redis前提の定番ジョブ基盤
- [solid_queue](../周辺インフラ/solid_queue.md) … DBバックエンド＝Redis不要のジョブ（Rails 8既定）
- （DBエンジン・Puma・S3 等も `../周辺インフラ/` に順次追加）

### リクエスト処理・設定
- [strong_parameters.md](./strong_parameters.md) … Strong Parameters とは
- [filters.md](./filters.md) … before_action などフィルタとは
- [partial_layout.md](./partial_layout.md) … partial / layout とは
- [session_cookie_flash.md](./session_cookie_flash.md) … session / cookie / flash とは
- [config_credentials.md](./config_credentials.md) … 設定 / credentials / ENV とは

### 認証・非同期・メール
- [auth.md](./auth.md) … 認証・認可とは
- [active_job.md](./active_job.md) … Active Job（バックグラウンドジョブ）とは
- [action_mailer.md](./action_mailer.md) … Action Mailer とは

### 設計・整理パターン
- [concern.md](./concern.md) … Concern とは
- [module.md](./module.md) … module（名前空間 / ミックスイン / Engine）
- [service_form.md](./service_form.md) … Service / Form オブジェクトとは

### アセット・テスト・安全・運用
- [assets.md](./assets.md) … アセット（importmap / Propshaft）とは
- [testing.md](./testing.md) … テストとは（RSpec / FactoryBot / system spec）
- [security.md](./security.md) … セキュリティとは（CSRF / XSS / SQLi）
- [console.md](./console.md) … rails console / コマンドとは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（早見表）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Rails 7）

## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
