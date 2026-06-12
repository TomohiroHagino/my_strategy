# 設定（settings.py / .env）（Django 5）

## ひとことで言うと
アプリの**環境ごとの挙動**（DB接続先・デバッグ可否・許可ホスト等）と、**秘密情報**（`SECRET_KEY`・APIキー・DBパスワード等）を一元管理する仕組み。前者は `settings.py`、後者は**環境変数（`.env` / OS環境）** で外から渡し、ソースに直書きしないのが鉄則。

## 役割・なぜ必要か
- 開発・テスト・本番で挙動を切り替えたい（例：本番だけ `DEBUG=False`、開発だけ詳細ログ）から、設定を分岐できる仕組みが要る。
- `SECRET_KEY` や DB パスワードをコードに直書きすると**漏洩事故**になる。環境変数で外から渡せば、リポジトリに平文の秘密を残さない。
- 「コードは Git 管理、秘密は別管理」を徹底するための仕組み。

## 基本の書き方（コード）
```python
# settings.py（startproject が生成する素のもの）
from pathlib import Path
BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = "django-insecure-..."   # ← 本来は秘密！環境変数へ逃がす（後述）
DEBUG = True                          # ← 本番では必ず False
ALLOWED_HOSTS = []                    # ← 本番では実ドメインを列挙

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "accounts",                       # 自作アプリはここに追加（忘れると認識されない）
]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "mydb", "USER": "myuser", "PASSWORD": "secret",  # ← 環境変数へ
        "HOST": "localhost", "PORT": "5432",
    }
}
```

### 環境変数で秘密を外出し（django-environ）
```python
# pip install django-environ
import environ
env = environ.Env(DEBUG=(bool, False))            # 型と既定値を宣言
environ.Env.read_env(BASE_DIR / ".env")           # .env を読み込む（無くても可）

SECRET_KEY = env("SECRET_KEY")                    # 無ければ起動時に例外＝早期発見
DEBUG = env("DEBUG")
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS", default=[])
DATABASES = {"default": env.db("DATABASE_URL")}    # URL一発でDB設定が組める
```
```bash
# .env（.gitignore に必ず入れる。コミット厳禁）
SECRET_KEY=本物のランダム文字列
DEBUG=False
ALLOWED_HOSTS=example.com,www.example.com
DATABASE_URL=postgres://myuser:secret@localhost:5432/mydb
```
標準ライブラリだけなら `python-dotenv` + `os.environ["SECRET_KEY"]` でも同じことができる。

### 設定分割（base / dev / prod）
```python
# settings/base.py … 共通設定
# settings/dev.py
from .base import *
DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]

# settings/prod.py
from .base import *
DEBUG = False
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")
SECURE_SSL_REDIRECT = True
```
```bash
# 使う設定モジュールを環境変数で指定
export DJANGO_SETTINGS_MODULE=config.settings.prod
python manage.py runserver  # dev を読みたいなら dev を指定
```

### SECRET_KEY を安全に生成
```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

## 実務での使い方・定番パターン
- **秘密は環境変数、純粋な設定値は settings**、と住み分ける。`SECRET_KEY` / DB パスワード / 外部 API キーは必ず env。
- **`settings/` ディレクトリ分割**（base/dev/prod）が定番。共通は base、差分だけ各環境に。`DJANGO_SETTINGS_MODULE` で切替。
- 本番ホスティング（Render / Railway / k8s Secret 等）では **OS 環境変数を注入**。`.env` は開発専用と割り切る。
- `env("...")` で**必須キーを宣言**しておくと、設定漏れが起動時に即エラーになり気付ける。
- `python manage.py check --deploy` で本番設定の抜け（`DEBUG`・SSL・Cookie 等）を自動チェックできる。

## ハマりどころ / アンチパターン
- **`SECRET_KEY` をコミット**（重大事故）。漏れたら**即ローテーション**（セッション・署名が偽造可能になる）。`django-insecure-` で始まる初期値は本番で使わない。
- **本番で `DEBUG=True`**（最重大）：例外時にソース・設定・環境変数が画面に丸見えになり、**深刻な情報漏洩**。本番は必ず `False`。
- **`ALLOWED_HOSTS` 未設定**：`DEBUG=False` だと全リクエストが 400 で弾かれて「本番で真っ白」になる。実ドメインを必ず列挙。
- **`.env` をコミット**：`.gitignore` に入れ忘れて秘密ごと公開、が定番事故。`.env.example`（値は空）だけ共有する。
- **`INSTALLED_APPS` への追加忘れ**：自作アプリやサードパーティを足し忘れて「モデル/テンプレートが認識されない」。
- **設定値の直書き（マジック値）**：URL やキーをコード散在 → settings か env に寄せる。

## 関連
[getting_started.md](./getting_started.md) / [security.md](./security.md) / [static_media.md](./static_media.md)
