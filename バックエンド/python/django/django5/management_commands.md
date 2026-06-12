# manage.py コマンド / shell（Django 5）

## ひとことで言うと
`manage.py` は、**プロジェクト設定（`DJANGO_SETTINGS_MODULE`）を読み込んだ状態で Django のCLIを実行する入口**。`migrate` / `runserver` / `shell` などの組込コマンドや、自作のカスタムコマンドをここから叩く。Rails の `rails` コマンドに相当する。

## 役割・なぜ必要か
- アプリ本体を読み込んだ状態で「DB操作」「サーバ起動」「対話シェル」「バッチ処理」を実行するためにある。
- `python manage.py shell` は **アプリを読み込んだ REPL**。モデルやサービスを実コードのまま呼び出し、データ確認・動作検証・調査ができる（Rails の `rails console` 相当）。
- 定期バッチや運用作業は、Viewやcronスクリプトに散らさず **カスタムコマンド（`BaseCommand`）にまとめる**のが定番。

## 基本の書き方（コード）
```bash
# 組込コマンド（よく使うもの）
python manage.py migrate              # マイグレーション適用（DBスキーマ反映）
python manage.py runserver            # 開発サーバ起動（既定 127.0.0.1:8000）
python manage.py runserver 0.0.0.0:8000  # 外部公開・ポート指定
python manage.py makemigrations       # モデル変更からマイグレーション生成
python manage.py shell                # 対話シェル（REPL）
python manage.py createsuperuser      # admin用の管理ユーザー作成
python manage.py collectstatic        # 静的ファイルを STATIC_ROOT へ集約（本番デプロイ時）
python manage.py test                 # テスト実行
python manage.py showmigrations       # マイグレーション適用状況の確認
```
```python
# python manage.py shell の中で（アプリが読み込まれた状態）
from myapp.models import User
User.objects.count()                  # 件数
u = User.objects.get(email="a@b.c")   # 取得
u.posts.filter(published=True)[:3]    # 関連・絞り込みもそのまま使える
# Django 5.2 では shell 起動時にモデルが自動 import される（手動 import 不要に）
```
```bash
# django-extensions を入れると shell_plus が使える（全モデル自動import）
pip install django-extensions
# settings.py の INSTALLED_APPS に "django_extensions" を追加
python manage.py shell_plus           # 全モデルを自動importしたREPL
python manage.py shell_plus --print-sql  # 実行されたSQLを表示（N+1調査に便利）
```
```python
# カスタムコマンド: myapp/management/commands/send_reminders.py
# ★ ディレクトリ構成が必須: <app>/management/commands/<name>.py
#   management/ と commands/ には空の __init__.py を置く
from django.core.management.base import BaseCommand
from myapp.models import Reminder

class Command(BaseCommand):
    help = "未送信のリマインダーを送信する"

    def add_arguments(self, parser):
        parser.add_argument("--dry-run", action="store_true", help="実送信せず件数だけ表示")

    def handle(self, *args, **options):
        qs = Reminder.objects.filter(sent=False)
        if options["dry_run"]:
            self.stdout.write(self.style.WARNING(f"対象 {qs.count()} 件（dry-run）"))
            return
        for r in qs:
            r.send()
        self.stdout.write(self.style.SUCCESS(f"{qs.count()} 件送信しました"))
```
```bash
python manage.py send_reminders --dry-run   # 自作コマンドの呼び出し
```

## 実務での使い方・定番パターン
- **バッチ処理＝カスタムコマンド**：cron や Celery Beat から `python manage.py <command>` を叩く形にすると、設定読み込み・引数処理・出力整形（`self.style`）が揃って扱いやすい。
- **`add_arguments` で引数を受ける**：`--dry-run` のようなフラグを付け、いきなり本実行せず確認できるようにする。
- **`self.stdout.write` / `self.style`**：`print` でなくこれを使うと、テスト時の出力捕捉や色付け（SUCCESS/WARNING/ERROR）が効く。
- **shell での調査**：`User.objects.filter(...)` をその場で試す、`shell_plus --print-sql` でN+1を発見する、といった日常調査に使う。→ [orm_queries.md](./orm_queries.md)
- **`migrate` / `makemigrations` の順序**：モデル変更 → `makemigrations`（差分ファイル生成）→ `migrate`（DBへ適用）。→ [migrations.md](./migrations.md)
- **プロジェクトの起点**：`startproject` / `startapp` での雛形生成は最初の一歩。→ [getting_started.md](./getting_started.md)

## ハマりどころ / アンチパターン
- **本番 shell での直接データ変更（最頻・危険）**：`delete()` / `update()` / `objects.all().delete()` は取り返しがつかない。本番では原則「読むだけ」、変更が必要なら**対象を `get`/`filter` で確定→件数確認→トランザクション内**で実行。理想はカスタムコマンド化してレビュー・dry-runを通す。
- **コマンドの置き場所ミス**：`<app>/management/commands/` 以外に置く、`__init__.py` を忘れる、ファイル名とコマンド名がずれる、と認識されない。`management/` と `commands/` 両方に空の `__init__.py` が必要。
- **`runserver` を本番で使う**：開発用サーバなので本番禁止。本番は gunicorn/uvicorn + nginx 等を使う。
- **`collectstatic` 忘れ**：本番で静的ファイルが404になる典型。デプロイ手順に含める。→ [static_media.md](./static_media.md)
- **`shell` でのモデル import 漏れ**：Django 5.2未満では自動importされない。`shell_plus` か手動 import を使う。
- **コマンド内で例外を握り潰す**：失敗を黙って飲むと「動いたつもり」になる。明示的にログ・終了コードで失敗を伝える。

## 関連
[getting_started.md](./getting_started.md) / [migrations.md](./migrations.md) / [orm_queries.md](./orm_queries.md)
