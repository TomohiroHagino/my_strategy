# Rails 5 実務リファレンス（索引）

> **この版 = Rails 5 系（5.0〜5.2）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。
> （Rails 7 版は別ディレクトリ `../rails7/`。周辺ミドルウェアは版共通の `../周辺インフラ/`。）

## この版のポイント（Rails 5 で何が変わったか）
- **API モード（`rails new --api`）** … ビュー無しでJSONだけ返す構成が公式サポート。
- **Action Cable（WebSocket）** … リアルタイム通信が標準同梱（Redis前提）。
- **`rails` コマンドに統一** … `rake db:migrate` ではなく `rails db:migrate` が使える（rake も残る）。
- **`ApplicationRecord` / `ApplicationJob` / `ApplicationMailer`** … 各基底クラスが導入され、アプリ共通設定の置き場ができた。
- **`belongs_to` が既定で必須**（presence 検証が自動で付く。任意にするなら `optional: true`）。**Attributes API** 導入。
- フロントは **Turbolinks 5 ＋ rails-ujs**（5.1で jquery_ujs からバニラJSの rails-ujs に置換）。**Hotwire は無い**。`*.js.erb`（SJR）もまだ現役。

## Rails 7 との主な違い（先に押さえる）
- フロントが **Turbolinks + rails-ujs（jQuery系）**。Turbo / Stimulus / importmap は無い。
- 削除リンクは **`method: :delete` + `data-confirm`**（`turbo_method` ではない）。
- フォーム再描画に `status: :unprocessable_entity` は不要（Turbo が無いため）。
- アセットは **Sprockets が既定**（Propshaft は無い）。Webpacker は 5.1 で選択肢として追加。
- 秘密情報は **5.0/5.1 は `secrets.yml`**、**5.2 で暗号化 credentials**（`credentials.yml.enc` + `master.key`）。

## 5.0 → 5.1 → 5.2 の追加（版内の差分）
- **5.0**：APIモード / Action Cable / `ApplicationRecord` 等 / `belongs_to` 必須化 / Attributes API / `rails` コマンド統一。
- **5.1**：rails-ujs（jquery_ujs を置換）/ **Webpacker（yarn）導入** / **system test（Capybara）標準同梱** / 暗号化 secrets / フォームの `form_with`。
- **5.2**：**暗号化 credentials**（`secrets.yml` から移行）/ **Active Storage**（ファイルアップロード標準）/ **bootsnap**（起動高速化）/ Content Security Policy DSL。

## リクエストの流れ（全体像）
```
ブラウザ → ルーティング(routes.rb) → コントローラ#アクション
        → モデル(Active Record)でDB操作 → ビュー(View)でHTML生成 → レスポンス
（Turbolinks ならページ遷移はAjax化＋body差し替え / APIモードならJSON）
```

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（`rails new`〜起動〜scaffold、`--api` も）

### コア（MVC）
- [request_flow.md](./request_flow.md) … リクエスト/処理の流れ・各層は何を返すか（全体像）
- [routing.md](./routing.md) … ルーティング
- [controller.md](./controller.md) … コントローラ（APIモードのレスポンス含む）
- [model.md](./model.md) … モデル（ApplicationRecord・belongs_to必須・Attributes API）
- [view.md](./view.md) … ビュー
- [helper.md](./helper.md) … ヘルパー
- [active_record.md](./active_record.md) … DB / Active Record（マイグレーション・関連・クエリ・N+1）

### フロント
- [javascript.md](./javascript.md) … RailsでのJSの書き方（rails-ujs + Turbolinks 5 + Webpacker任意。タスク別具体例）
- [assets.md](./assets.md) … アセット（Sprockets既定＋Webpacker 5.1選択）

### リクエスト処理・設定
- [strong_parameters.md](./strong_parameters.md) … Strong Parameters
- [filters.md](./filters.md) … before_action などフィルタ
- [partial_layout.md](./partial_layout.md) … partial / layout
- [session_cookie_flash.md](./session_cookie_flash.md) … session / cookie / flash
- [config_credentials.md](./config_credentials.md) … 設定 / credentials / ENV（5.2 credentials、5.0/5.1 は secrets.yml）

### 認証・非同期・メール
- [auth.md](./auth.md) … 認証・認可
- [active_job.md](./active_job.md) … Active Job（ApplicationJob・Sidekiq）
- [action_mailer.md](./action_mailer.md) … Action Mailer

### Rails 5 特有
- [api_mode.md](./api_mode.md) … APIモード（`--api`）
- [action_cable.md](./action_cable.md) … Action Cable（WebSocket / Redis）
- [active_storage.md](./active_storage.md) … Active Storage（5.2 のファイルアップロード）

### 設計・整理パターン
- [concern.md](./concern.md) … Concern
- [module.md](./module.md) … module（名前空間 / ミックスイン / Engine）
- [service_form.md](./service_form.md) … Service / Form オブジェクト

### テスト・安全・運用
- [testing.md](./testing.md) … テスト（Minitest / RSpec、system test 5.1）
- [security.md](./security.md) … セキュリティ（CSRF / XSS / SQLi）
- [console.md](./console.md) … rails console / コマンド
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（belongs_to必須化など）

### 関連事項（周辺インフラ）→ 版共通の [`../周辺インフラ/`](../周辺インフラ/) に集約
- [redis](../周辺インフラ/redis.md) … キャッシュ / ジョブ / Action Cable の土台
- [sidekiq](../周辺インフラ/sidekiq.md) … Redis前提の定番ジョブ基盤
- [unicorn](../周辺インフラ/unicorn.md) … Rails 5 時代によく使われたアプリサーバ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Rails 5）

## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
