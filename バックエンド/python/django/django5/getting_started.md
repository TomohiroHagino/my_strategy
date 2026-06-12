# 始め方（getting started）（Django 5）

## ひとことで言うと
Django 5 のプロジェクトを**ゼロから動かすまでの一連の手順**。仮想環境作成 → インストール → `startproject` → `startapp` → アプリ登録 → `migrate` → `runserver`、までを一直線で通す。

## 役割・なぜ必要か
- Django は「設定の集合体（project）」＋「機能の集合体（app）」で成り立つので、**最初に骨組みを正しく作る**と後がラク。
- Python 製のため **仮想環境（venv）でプロジェクトごとに依存を隔離**するのが大前提。グローバルに入れると複数プロジェクトでバージョンが衝突する。→ venv / conda / Docker の棲み分けは [環境管理（図解）](../../環境管理.md)
- Django 5 は **Python 3.10 以上が必須**（5.2 LTS も同様）。古い Python では `pip install` が通っても動かない。
- MTV なので「View＝処理（他FWのコントローラ役）」「Template＝見た目」。この呼び替えだけ先に頭へ。→ [project_apps.md](./project_apps.md)

## 基本の書き方（コード）
```bash
# 1) 仮想環境を作って有効化（プロジェクト直下で）
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
# プロンプト先頭が (.venv) になればOK

# 2) Django をインストール（5系を明示）
python -m pip install --upgrade pip
pip install "Django>=5.2,<6.0"
python -m django --version          # 5.2.x が出ればOK

# 3) プロジェクト（全体設定）を作る
django-admin startproject mysite .  # 末尾の . でカレントに展開（mysite/mysite/ の二重を避ける）

# 4) アプリ（機能単位）を作る
python manage.py startapp blog

# 5) 初回マイグレーション（admin/auth等の組み込みテーブルを作成）
python manage.py migrate

# 6) 管理ユーザー作成（admin ログイン用）
python manage.py createsuperuser

# 7) 開発サーバー起動
python manage.py runserver          # http://127.0.0.1:8000/
```

```python
# mysite/settings.py … 作った app を必ず登録する
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "blog",                          # ← 追加（または "blog.apps.BlogConfig"）
]
```

ディレクトリ構成（`startproject mysite . ＋ startapp blog` の結果）:
```
.
├── manage.py            # 各種コマンドの入口（runserver / migrate など）
├── mysite/              # project（全体設定）
│   ├── settings.py      # 設定本体（INSTALLED_APPS / DB / etc.）
│   ├── urls.py          # ルートのURLconf
│   ├── wsgi.py / asgi.py# 本番サーバー用エントリ
│   └── __init__.py
└── blog/                # app（機能単位）
    ├── models.py        # モデル（ORM）
    ├── views.py         # View（処理）
    ├── admin.py         # 管理画面の登録
    ├── apps.py          # アプリ設定（BlogConfig）
    ├── migrations/      # マイグレーション置き場
    └── tests.py
```

## 実務での使い方・定番パターン
- **依存は `requirements.txt`（or `pyproject.toml`）に固定**。`pip freeze > requirements.txt` で再現性を担保。チームは `pip install -r requirements.txt` で揃える。
- **`startproject mysite .`（末尾ドット）** をクセにする。ドット無しだと `mysite/mysite/` の入れ子になり迷子になりやすい。
- `runserver` は**開発専用**。本番は Gunicorn/Uvicorn＋Nginx 等を使う（`runserver` を本番で使わない）。
- モデルを足したら **`makemigrations` → `migrate`** の2段（`migrate` 単体ではモデル変更は反映されない）。→ [migrations.md](./migrations.md)
- `.env` で秘密情報（`SECRET_KEY`/DB接続）を外出しし、`settings.py` から読む。→ [settings.md](./settings.md)
- `.gitignore` に `.venv/` `*.pyc` `__pycache__/` `db.sqlite3` `.env` を入れる。

## ハマりどころ / アンチパターン
- **venv 未有効化**：`(.venv)` が出ていないまま `pip install` → グローバルに入って「動くマシンと動かないマシン」が発生。まず `source .venv/bin/activate`。
- **INSTALLED_APPS 登録漏れ**：app を作っただけでは認識されない。`makemigrations blog` が「No changes detected」になる典型原因。`INSTALLED_APPS` に追加。→ [project_apps.md](./project_apps.md)
- **`migrate` 忘れ**：起動はするが `no such table: ...` で落ちる。初回は必ず `migrate`、モデル変更後は `makemigrations` → `migrate`。
- **`python` と `python3` の取り違え**：venv 有効化後は `python` でOK。混在させると別環境を叩く。
- **`django-admin` を venv 外で実行**：バージョンが食い違う。常に有効化した venv 内で。
- **Python 3.9 以下で 5系を使おうとする**：インストール時/起動時に弾かれる。3.10+ を用意。

## フォルダ構成（始動直後）
```
mysite/                                   # ← startproject で作る「プロジェクト」
├── manage.py                             # CLI（runserver/migrate/startapp …）【生成】
├── mysite/                               # 設定パッケージ【生成】
│   ├── __init__.py
│   ├── settings.py                       # 全設定（INSTALLED_APPS, DB …）
│   ├── urls.py                           # ルートURLconf
│   └── asgi.py  wsgi.py                  # 配信エントリ
├── myapp/                                # ← startapp で作る「アプリ（機能単位）」
│   ├── __init__.py
│   ├── apps.py                           # アプリ設定【生成】
│   ├── admin.py                          # 管理画面登録【生成】
│   ├── models.py                         # モデル=テーブル定義【生成・自分で書く】
│   ├── views.py                          # ビュー=処理【生成・自分で書く】
│   ├── tests.py                          # テスト【生成】
│   ├── migrations/
│   │   ├── __init__.py                   # 【生成】
│   │   └── 0001_initial.py               # makemigrations で生成
│   ├── urls.py                           # アプリ内ルーティング（自分で作る）
│   ├── forms.py                          # フォーム（自分・任意）
│   ├── serializers.py                    # DRFのシリアライザ（自分・DRF時）
│   ├── templates/myapp/index.html        # テンプレート（自分で作る）
│   └── static/myapp/                     # 静的ファイル（自分で作る）
├── templates/                            # プロジェクト共通テンプレ（自分・任意）
├── static/                               # 収集先 collectstatic（自分）
├── requirements.txt                      # 依存（自分で作る）
├── db.sqlite3                            # SQLite（migrate 後に生成）
└── .env                                  # 環境変数（自分・django-environ 等）
```
- **「プロジェクト（mysite＝設定）」と「アプリ（myapp＝機能）」が分離**。1プロジェクトに複数アプリを持てる。
- **【生成】= `django-admin startproject` / `manage.py startapp`。** `urls.py`(アプリ)・`templates/`・`static/`・`forms.py`・`serializers.py` は自分で足す。
- ルートの `urls.py` から各アプリの `urls.py` を `include()` で繋ぐ。

## 関連
[project_apps.md](./project_apps.md) / [settings.md](./settings.md)
