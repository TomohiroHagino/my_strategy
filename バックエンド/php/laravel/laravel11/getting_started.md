# プロジェクトの始め方（Laravel 11）

## ひとことで言うと
`composer` でLaravelプロジェクトを生成し、`.env` 設定・`APP_KEY` 生成・マイグレーションを済ませて `php artisan serve` で動かすまでの**初期セットアップ一式**。

## 役割・なぜ必要か
- Laravelは「フレームワーク本体＋大量の規約（ディレクトリ構成・命名）」で成り立つ。最初に**正しい骨格を生成**しておくことで、以降の `make:*` コマンドやオートロードが規約どおりに効く。
- `.env` と `APP_KEY` は**暗号化・セッション・設定の前提**。ここが欠けると暗号化系がすべて落ちる。
- Laravel 11 は**スリム骨格**になった。`app/Http/Kernel.php` / `app/Console/Kernel.php` が廃止され、ミドルウェア・例外・ルーティング登録は **`bootstrap/app.php` に集約**された。古い記事の「Kernel.phpに登録」は11では通用しない。

## 基本の書き方（コード）
```bash
# 方法A: Laravelインストーラ（事前に composer global require laravel/installer が必要）
laravel new myapp

# 方法B: composer 単体で生成（インストーラ不要・実務で確実）
composer create-project laravel/laravel myapp

cd myapp

# 依存とフロント資産（Vite）
npm install

# .env は composer 生成時に自動コピーされるが、無ければ手動で
cp .env.example .env

# APP_KEY 生成（暗号化・セッションの前提。未生成だと例外）
php artisan key:generate

# DB作成（Laravel 11 既定は SQLite。ファイルが無ければ作る）
touch database/database.sqlite
php artisan migrate

# 開発サーバ起動（http://127.0.0.1:8000）
php artisan serve

# 別ターミナルでフロント資産をビルド監視（Blade内の @vite が参照）
npm run dev
```

```bash
# .env の主要項目（SQLite既定）。MySQLにするなら下記コメント側へ
APP_NAME=myapp
APP_KEY=base64:...          # key:generate が自動で埋める
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=sqlite
# DB_CONNECTION=mysql
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=myapp
# DB_USERNAME=root
# DB_PASSWORD=secret
```

```bash
# よく使う生成コマンド
php artisan make:model Post -mcr   # モデル + マイグレーション(-m) + コントローラ(-c) + resource(-r)
php artisan make:controller PostController
php artisan make:migration create_posts_table
```

## 実務での使い方・定番パターン
- **ディレクトリ構成（11スリム骨格）**:
  - `app/` … `Models/` `Http/Controllers/` `Providers/`（Kernelは無い）
  - `routes/` … `web.php` / `console.php`（API は後述の `install:api` で `api.php` 追加）
  - `resources/views/` … Bladeテンプレート、`resources/js` `resources/css` … Vite対象
  - `database/migrations/` `database/seeders/` `database/factories/`
  - `config/` … 11では**最小**。必要な設定だけ `php artisan config:publish` で出す
  - `bootstrap/app.php` … **ミドルウェア・例外・ルーティングの登録ハブ**（旧Kernelの役割）
- **新規参画時の儀式**: `composer install` → `cp .env.example .env` → `php artisan key:generate` → `php artisan migrate` の順。これでだいたい動く。
- **APIを使うなら**: 既定では `routes/api.php` が無い。`php artisan install:api` を実行して導入する。→ [routing.md](./routing.md)
- `php artisan migrate:fresh --seed` で**DBを作り直して初期データ投入**。開発中の定番。

## ハマりどころ / アンチパターン
- **`APP_KEY` 未生成**: 「No application encryption key has been specified.」で落ちる。`php artisan key:generate` を忘れない。
- **PHPバージョン不足**: Laravel 11 は **PHP 8.2 以上必須**。`php -v` で確認。古いPHPだと `composer install` 時に弾かれる。
- **storage / bootstrap/cache の権限**: Webサーバ経由で動かすとき書き込み権限が無いと500。`chmod -R 775 storage bootstrap/cache`（所有者はWebサーバユーザに）。
- **DB接続エラー**: SQLite既定なのに `database/database.sqlite` を作り忘れて `migrate` が落ちる。MySQLに変えたら `.env` の `DB_*` を直して `php artisan config:clear`。
- **`config(...)` が `.env` を読まない**: 本番で `config:cache` 済みだと `.env` 変更が反映されない。`php artisan config:clear` または再キャッシュ。`.env` をコード中で `env()` 直読みするのもアンチパターン（cache後にnullになる）→ `config()` 経由にする。→ [config_env.md](./config_env.md)
- **古い記事の `Kernel.php` 手順をそのまま実行**: 11には無い。ミドルウェア登録は `bootstrap/app.php` で行う。→ [middleware.md](./middleware.md)

## フォルダ構成（始動直後）
```
myapp/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Controller.php            # 基底コントローラ【生成】
│   │   │   └── UserController.php        # 自分で作る（php artisan make:controller）
│   │   ├── Requests/
│   │   │   └── StoreUserRequest.php      # フォームリクエスト=バリデーション（自分）
│   │   ├── Resources/
│   │   │   └── UserResource.php          # APIレスポンス整形（自分）
│   │   └── Middleware/                   # カスタムMW（自分・登録は bootstrap/app.php）
│   ├── Models/
│   │   └── User.php                      # Eloquentモデル【生成】
│   ├── Providers/
│   │   └── AppServiceProvider.php        # サービスプロバイダ【生成】
│   └── Services/                         # 業務ロジック（自分・任意）
├── bootstrap/
│   ├── app.php                           # 起動・ルーティング/MW/例外設定【生成・L11の要】
│   ├── providers.php                     # プロバイダ登録【生成】
│   └── cache/                            # 起動キャッシュ
├── config/                               # 各種設定 app.php/database.php …【生成】
├── database/
│   ├── migrations/                       # マイグレーション（users等）【生成】
│   ├── seeders/DatabaseSeeder.php        # 初期データ【生成】
│   ├── factories/UserFactory.php         # テスト用ダミー生成【生成】
│   └── database.sqlite                   # SQLite選択時のDB実体
├── public/
│   ├── index.php                         # 公開エントリ（全リクエストの入口）【生成】
│   └── .htaccess
├── resources/
│   ├── views/welcome.blade.php           # Bladeテンプレ【生成】
│   └── css/app.css  js/app.js            # フロント資産【生成】
├── routes/
│   ├── web.php                           # Webルーティング【生成】
│   ├── console.php                       # コマンド/スケジュール【生成】
│   └── api.php                           # APIルート（php artisan install:api で追加）
├── storage/                              # app/・framework/・logs/（ログ・キャッシュ）
├── tests/
│   ├── Feature/                          # 機能テスト【生成】
│   └── Unit/                             # 単体テスト【生成】
├── artisan                               # CLI（make/migrate/serve …）【生成】
├── composer.json                         # PHP依存【生成】
├── package.json / vite.config.js         # フロントビルド（Vite）【生成】
├── .env / .env.example                   # 環境変数【生成】
└── .gitignore
```
- **【生成】= `laravel new` が作る。** `Controllers`/`Requests`/`Resources`/`Services` の具体クラスは `php artisan make:*` で自分で足す。
- **Laravel 11 はスリム化**：`app/Http/Kernel.php`・`app/Console/Kernel.php` は廃止され、MW・例外・スケジュールは **`bootstrap/app.php`** に集約。`api.php` は既定で無く `php artisan install:api` で追加。
- 入口は `public/index.php`、ルートは `routes/web.php`、ビューは `resources/views`（Blade）。

## 関連
[routing.md](./routing.md) / [database.md](./database.md) / [config_env.md](./config_env.md)
