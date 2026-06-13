# CodeIgniter 4 実務リファレンス（索引）

> **この版 = CodeIgniter 4（PHP 8.1+）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> CI3 からの全面刷新（名前空間・Composer・`spark` CLI・PSR準拠）が前提。

## この版のポイント（CI4 で変わったこと）
- **名前空間 / Composer / PSR-4 オートロード**（CI3の手動include文化から脱却）。
- **`spark` CLI**（`php spark serve` / `make:*` / `migrate`）。
- **Query Builder ＋ Model**（バリデーション・自動タイムスタンプ等を内蔵）。
- **Filters**（before/after の共通処理＝ミドルウェア相当）。
- 認証は公式パッケージ **Shield**。

## リクエストの流れ
```
ブラウザ → Routes(app/Config/Routes.php) → Controller
        → Model(Query Builder)でDB操作 → View でHTML生成 → レスポンス
（共通処理は Filters が前後に挟まる）
```

## 項目（各ファイルへ）

### はじめに / MVC
- [getting_started.md](./getting_started.md) … 始め方（composer / spark serve / 構成）
- [ハンズオン.md](./ハンズオン.md) … 手を動かす実習（0からコントローラを1つ動かす／404・baseURLを直す）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [routing.md](./routing.md) … ルーティング（Routes / 自動ルーティング）とは
- [controllers.md](./controllers.md) … コントローラとは
- [models.md](./models.md) … モデル（CI4 Model）とは
- [views.md](./views.md) … ビュー（view() / レイアウト / View Cells）とは
- [request_response.md](./request_response.md) … リクエスト / レスポンスとは

### データ・検証・横断
- [database_query_builder.md](./database_query_builder.md) … DB / Query Builder / マイグレーションとは
- [validation.md](./validation.md) … バリデーションとは
- [filters.md](./filters.md) … フィルタ（before/after＝ミドルウェア相当）とは
- [config_env.md](./config_env.md) … 設定（Config クラス / .env）とは
- [sessions.md](./sessions.md) … セッションとは

### 認証・安全・テスト
- [auth_shield.md](./auth_shield.md) … 認証（Shield）とは
- [security.md](./security.md) … セキュリティ（CSRF / XSS / エスケープ）とは
- [testing.md](./testing.md) … テスト（PHPUnit / CIUnitTestCase）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（CodeIgniter 4）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
