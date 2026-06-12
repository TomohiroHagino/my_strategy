# 設定 / .env（Laravel 11）

## ひとことで言うと
**`.env`** = 環境（ローカル/本番）ごとに変わる値を入れるファイル。**`config/`** = アプリ設定の本体。コードからは `config()` ヘルパで読む。

## 役割・なぜ必要か
- DB接続先・APIキー・デバッグ可否などは**環境ごとに違う**。これらをコードに直書きせず `.env` に逃がし、Gitに含めない（`.env.example` だけ共有）ことで、**秘密情報の漏洩を防ぎ**、環境ごとの差し替えを容易にする。
- `.env` の値は基本 `config/*.php` を経由して使う。**コード本体から `env()` を直接呼ぶのは原則NG**（理由は後述の `config:cache`）。アプリは `config()` を、`config/` の中だけが `env()` を読む、という二層構造。
- **Laravel 11 は `config/` を最小化**：既定では設定ファイルがほとんど無く（フレームワーク内蔵値を使う）、上書きしたいものだけ `php artisan config:publish` で取り出す方針に変わった。

## 基本の書き方（コード）
```ini
# .env （Gitに入れない。.env.example を雛形として共有）
APP_NAME=MyApp
APP_ENV=local
APP_KEY=base64:....                 # 暗号化キー（必須）
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=sqlite                # Laravel 11 既定
SESSION_DRIVER=database
CACHE_STORE=database

STRIPE_KEY=sk_test_xxx              # 例：外部APIキー
```

```php
// config/services.php （config の中だけが env() を読む）
return [
    'stripe' => [
        'key' => env('STRIPE_KEY'),          // ← env() はここで使う
    ],
];
```

```php
// アプリのコード：env() ではなく config() を使う
$key = config('services.stripe.key');        // 正しい
$name = config('app.name');
config(['app.timezone' => 'Asia/Tokyo']);    // 実行時の上書き（一時的）
```

```bash
# 11では config が最小。必要な設定ファイルだけ取り出して編集する
php artisan config:publish              # 対話で選択
php artisan config:publish app          # 特定のものだけ
```

## 実務での使い方・定番パターン
- **本番では設定をキャッシュ**：`php artisan config:cache` で全 `config/*.php` を1ファイルに固めて高速化する。デプロイ手順に組み込むのが定番。
  ```bash
  php artisan config:cache    # 本番デプロイ時
  php artisan config:clear    # 設定を変えたら作り直す
  ```
- **APP_KEY は最初に生成**：`php artisan key:generate`。これが無いと暗号化（Cookie/セッション）が動かない。
- **`.env` は環境ごとに**：`.env`（ローカル）／本番はサーバの環境変数や `.env.production` 相当。秘密は Secrets Manager 等に置き、`.env` を Git にコミットしない（`.gitignore` 済み）。
- **新しい設定の追加手順**：`.env` にキー追加 → 対応する `config/xxx.php` で `env('KEY')` を読む → コードからは `config('xxx.key')`。env() をコードに直書きしないこのルートを徹底する。
- **値の確認**：`php artisan tinker` で `config('app.name')`、`php artisan about` で主要設定の概要を一覧。

## ハマりどころ / アンチパターン
- **`config:cache` 後にコード内で `env()`**：設定をキャッシュすると、**`config/` の外で呼んだ `env()` は `null` を返す**。「ローカルでは動くのに本番だけ設定が空」の最頻原因。**コード本体は必ず `config()` 経由**にし、`env()` は `config/*.php` の中だけで使う。
- **APP_KEY 未設定**：`No application encryption key has been specified.` で落ちる。`php artisan key:generate` を実行。`.env` をコピーして始めた時に忘れがち。
- **`.env` を変えたのに反映されない**：`config:cache` 済みだと `.env` 変更が効かない。`php artisan config:clear`（または再 `config:cache`）が必要。
- **`.env` を Git にコミット**：APIキー・DBパスワードが漏洩。`.env` は除外、`.env.example` だけ共有が鉄則。漏れたキーは即ローテーション。
- **11なのに `config/` を探して無い**：最小化されたため。`php artisan config:publish` で必要なファイルを出してから編集する（古い記事は最初から全ファイルある前提で書かれている）。

## 関連
[getting_started.md](./getting_started.md) / [session_cache.md](./session_cache.md) / [security.md](./security.md)
