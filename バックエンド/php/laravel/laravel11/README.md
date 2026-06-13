# Laravel 11 実務リファレンス（索引）

> **この版 = Laravel 11（2024〜、PHP 8.2+ 必須）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。

## この版のポイント（Laravel 11 で何が変わったか）
- **スリム化された骨格**: `app/Http/Kernel.php` / `app/Console/Kernel.php` が廃止。**`bootstrap/app.php` に集約**（ミドルウェア・例外・ルーティング登録をここで設定）。
- **デフォルトSQLite**: `.env` 既定がSQLite（セッション/キャッシュ/キューもDB既定に寄せた）。
- **`config/` が最小**: 設定ファイルは既定で減り、必要なものだけ `php artisan config:publish` で出す。
- **health ルート**（`/up`）標準、**per-secondレートリミット**、**graceful なキャスト** など。
- **Reverb**（自前WebSocketサーバ）登場。Laravel 12 は 11 の延長（骨格は同じ・依存更新中心）。

## リクエストの流れ（全体像）
```
ブラウザ → routes/web.php(or api.php) → ミドルウェア → コントローラ
        → Eloquent でDB操作 → Blade でHTML生成 → レスポンス
（DI: Service Container が依存を解決して各所に注入）
```

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（composer / artisan serve / migrate）
- [ハンズオン.md](./ハンズオン.md) … 手を動かす実習（0からJSONを返す／APP_KEY未設定・migrate前を直す）

### コア（MVC）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [routing.md](./routing.md) … ルーティングとは
- [controller.md](./controller.md) … コントローラとは
- [model.md](./model.md) … モデル（Eloquent）とは
- [blade.md](./blade.md) … Blade（ビュー/テンプレート/コンポーネント）とは
- [database.md](./database.md) … DB（マイグレーション/クエリビルダ/リレーション/N+1）とは

### Laravelの土台（ここがRailsと違う肝）
- [service_container.md](./service_container.md) … サービスコンテナ（DI）とは
- [service_providers.md](./service_providers.md) … サービスプロバイダとは
- [facades.md](./facades.md) … ファサードとは

### リクエスト処理・設定
- [middleware.md](./middleware.md) … ミドルウェアとは
- [validation.md](./validation.md) … バリデーション / フォームリクエストとは
- [session_cache.md](./session_cache.md) … セッション / Cookie / キャッシュとは
- [config_env.md](./config_env.md) … 設定 / .env とは

### 認証・非同期・メール
- [auth.md](./auth.md) … 認証・認可（Gate / Policy / Sanctum）とは
- [queues.md](./queues.md) … キュー / ジョブ（Horizon）とは
- [mail.md](./mail.md) … メール（Mailable）/ 通知とは

### CLI・アセット・テスト・安全・運用
- [artisan.md](./artisan.md) … Artisan / Tinker とは
- [assets.md](./assets.md) … アセット（Vite）とは
- [testing.md](./testing.md) … テスト（Pest / PHPUnit）とは
  - [pest.md](./pest.md) … Pest（it/test・expect・datasets・Laravelプラグイン）とは
- [security.md](./security.md) … セキュリティ（CSRF / mass assignment / XSS / SQLi）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（早見表）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Laravel 11）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
