# project と app の構造（Django 5）

## ひとことで言うと
Django は **project（プロジェクト＝全体設定の入れ物）** と **app（アプリ＝機能単位の再利用モジュール）** の2層構造。1つの project に複数の app をぶら下げて作る。

## 役割・なぜ必要か
- **project** ＝サイト全体の設定。`settings.py`（DB・INSTALLED_APPS・ミドルウェア等）/ `urls.py`（ルートのURLconf）/ `wsgi.py`・`asgi.py`（本番サーバーの入口）を持つ。**動作の中身は持たず、束ねる役**。
- **app** ＝「ブログ」「決済」「ユーザー管理」のような**機能のかたまり**。`models.py`（ORM）/ `views.py`（処理）/ `admin.py` / `migrations/` / `apps.py`（アプリ設定）を持つ。
- 分けることで **再利用と責務分離**ができる。よく出来た app は別プロジェクトへ持ち運べる（`django.contrib.auth` 等の組み込み app もこの形）。
- MTV なので app の中心は **Model（データ＋業務ルール）＋View（処理）＋Template（見た目）**。「View＝コントローラ役」に注意。→ [views.md](./views.md)

## 基本の書き方（コード）
```bash
# project は最初に1回（startproject）
django-admin startproject mysite .

# app は機能ごとに作る（複数OK）
python manage.py startapp blog
python manage.py startapp accounts
```

```python
# mysite/settings.py … 作った app を必ず登録
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "blog",                      # ← 追加（短縮形）
    "accounts.apps.AccountsConfig",  # ← フル指定形（どちらでも可）
]
```

```python
# blog/apps.py … app のメタ設定（startapp が自動生成）
from django.apps import AppConfig


class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"

    def ready(self):
        # シグナル登録などはここで（import 副作用を集約）
        from . import signals  # noqa: F401
```

```
mysite/            # project（全体設定）= 中身を持たず束ねる
├── settings.py    #   設定本体
├── urls.py        #   ルートURLconf（各appのurls.pyをinclude）
├── wsgi.py/asgi.py
blog/              # app（機能単位）= 動作の中身
├── models.py / views.py / admin.py
├── apps.py        #   BlogConfig
├── migrations/    #   このapp固有のマイグレーション
└── urls.py        #   （任意）app内ルート
accounts/          # app（別機能）= もう1つの中身
└── ...
```

## 実務での使い方・定番パターン
- **app は「凝集した機能単位」で切る**。`blog` / `accounts` / `payments` のように。1機能＝1app が目安。
- **project名と app名を被らせない**（`mysite` の中に `mysite` app を作らない）。import が混乱する。
- 各 app に **`urls.py` を置き、project の `urls.py` から `include("blog.urls")`** で取り込むと URL もモジュール化できる。→ [urls.md](./urls.md)
- **`apps.py` の `ready()`** はシグナル登録の定番置き場。`import` 副作用をここに集約すると循環importを避けやすい。→ [signals.md](./signals.md)
- 共通の基盤（抽象モデル・ユーティリティ）は `core` app に寄せ、他 app から参照する設計が安定。
- **app 内はさらにファイル分割可**（`views/` パッケージ化など）。「多くの小さいファイル＞少数の巨大ファイル」。

## ハマりどころ / アンチパターン
- **app 未登録で動かない**：`INSTALLED_APPS` に入れ忘れると、モデルが認識されず `makemigrations` が「No changes detected」、テンプレートも `app_dirs` で読まれない。まず登録を疑う。→ [getting_started.md](./getting_started.md)
- **循環import**：`accounts.models` が `blog.models` を import し合うと起動時に落ちる。**文字列参照（`"blog.Post"`）の ForeignKey** や `apps.py` の `ready()` 内 import で回避。
- **app の粒度ミス**：何でも1 app に詰める巨大 app → 再利用・テスト不能。逆に小さく割りすぎても import 地獄。**機能境界で割る**。
- **app間の密結合**：他 app の内部実装に直接触る → 片方の変更で連鎖崩壊。**公開API（関数・シグナル）越し**にやり取りする。
- **`startapp` 後に `apps.py` を消す/名前変更**して `INSTALLED_APPS` のフル指定と食い違う → `ImportError`。短縮形かフル指定かを統一。

## 関連
[getting_started.md](./getting_started.md) / [settings.md](./settings.md)
