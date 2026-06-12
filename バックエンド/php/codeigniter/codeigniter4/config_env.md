# 設定（Config クラス / .env）（CodeIgniter 4）

## ひとことで言うと
- **設定 = アプリの動作を決める値** を、コードと分けて管理する仕組み。
- CI4 では **`app/Config/*.php` の Config クラス**（PHP のプロパティ）と、**`.env`（環境変数ファイル）** の2層で管理する。
- 環境（開発 / 本番）の違いは **`CI_ENVIRONMENT`** で切り替える。

## 役割・なぜ必要か
- DB 接続先・APIキー・デバッグ表示などは **環境ごとに違う**。コードに直書きすると、開発と本番で書き換えが必要になり事故る。
- `.env` に「その環境固有の値・秘密情報」を置き、Git にコミットしない（`.gitignore` 済み）ことで、**秘密をリポジトリから守る**。
- Config クラスは「型のあるデフォルト値の集約場所」。`.env` はそれを**環境ごとに上書き**する層、という役割分担。

## 基本の書き方（コード）
### Config クラス（app/Config/App.php など）
```php
<?php
namespace Config;
use CodeIgniter\Config\BaseConfig;

class App extends BaseConfig
{
    public string $baseURL = 'http://localhost:8080/';
    public string $indexPage = 'index.php';
    public string $defaultLocale = 'ja';
}
```
利用側はサービス経由で取得する。
```php
$config = config('App');          // Config\App のインスタンス（シングルトン）
echo $config->baseURL;
echo config('App')->defaultLocale; // 直接呼びでもOK
```

### .env（プロジェクト直下。env.example をコピーして作る）
```bash
# env.example をコピーして .env を作る（.env は Git 管理外）
cp env.example .env
```
```bash
# .env の中身（KEY = VALUE 形式。コメントは #）
CI_ENVIRONMENT = development

# Config クラスのプロパティを「クラス名.プロパティ名」で上書きできる
app.baseURL = 'http://localhost:8080/'
app.defaultLocale = 'ja'

# DB 設定（database.default.* で Config\Database の default 配列を上書き）
database.default.hostname = localhost
database.default.database = my_app
database.default.username = root
database.default.password = secret
database.default.DBDriver = MySQLi

# 独自の値（任意のキーも置ける）
STRIPE_SECRET_KEY = sk_test_xxx
```

### env() / getenv() で読む
```php
// env() … .env の値を取得。第2引数はデフォルト（未設定時の値）
$key = env('STRIPE_SECRET_KEY', '');

// getenv() … PHP標準。env() は内部でこれをラップしている
$envName = getenv('CI_ENVIRONMENT');

// 重要：Config プロパティと結びついた値は、コードで env() を直接呼ばず
// config('App')->baseURL のように Config 経由で読むのが定石
```

### Registrar（複数 Config を一括で上書き）
```php
<?php
// app/Config/Registrar.php に置くと、起動時に各 Config をまとめて拡張できる
namespace Config;

class Registrar
{
    // メソッド名 = Config クラス名。返り値の配列がプロパティにマージされる
    public static function App(): array
    {
        return [
            'CSPEnabled' => true,           // App::$CSPEnabled を上書き
        ];
    }
}
```

## 実務での使い方・定番パターン
- **秘密情報は必ず `.env`**（APIキー・DBパスワード・トークン）。Config クラスには「公開してよい既定値」だけ書く。
- **環境ごとの切り替えは `CI_ENVIRONMENT`** に集約。`development` ならエラー詳細表示・デバッグツールバー、`production` なら最小限の表示。
- `.env` の `app.baseURL` を環境ごとに変える（localhost / 本番ドメイン）。
- DB 接続は `Config\Database` を直接編集せず、**`database.default.*` を `.env` で上書き**するのが定番。
- 起動時に必須環境変数の存在チェックを入れる（無ければ例外で fail fast）。
```php
// 例：起動時バリデーション（サービスやブートで）
if (env('STRIPE_SECRET_KEY', '') === '') {
    throw new \RuntimeException('STRIPE_SECRET_KEY が未設定です');
}
```

## ハマりどころ / アンチパターン
- **`.env` を Git にコミットしてしまう（最重大）**：秘密がリポジトリ履歴に残る。`.gitignore` に `.env` があることを確認し、コミットしたら**秘密を即ローテーション**。共有は `env.example`（値を伏せた雛形）で行う。
- **本番で `CI_ENVIRONMENT=production` にし忘れる（重大）**：`development` のままだと**スタックトレースや詳細エラーがそのまま画面に出て情報漏洩**する。本番は必ず `production`。Web サーバ環境変数（`SetEnv CI_ENVIRONMENT production`）での設定も可。
- **`.env` の値がなぜか反映されない**：書式ミス（`=` の前後スペースは可だが、`KEY: VALUE` は不可）、または **`database.default.password`** のような「Config 配列の階層」を正しいキーで書けていない。`クラス名.キー = 値` の形を守る。
- **`env()` の値を信用しすぎる**：すべて文字列で返る。`'false'` は真の文字列。真偽は明示的に変換・比較する。
- **Config を `new` で生成**：毎回別インスタンスになり `.env` の上書きが効かないことがある。必ず `config('App')` / サービス経由で取得する。
- **本番で `writable/` や `.env` を公開ディレクトリに置く**：`public/` のみ公開し、`.env` はドキュメントルート外に置く。

## 関連
[getting_started.md](./getting_started.md) / [security.md](./security.md) / [sessions.md](./sessions.md)
