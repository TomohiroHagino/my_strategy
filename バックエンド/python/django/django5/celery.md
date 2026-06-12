# Celery（非同期タスク）（Django 5）

## ひとことで言うと
**Celery = Python の分散タスクキュー（Djangoとは別のサードパーティ製）**。重い/遅い処理を **リクエストの外（別プロセスのworker）** で非同期に実行する。Django には標準のジョブキューが無いため、非同期処理の事実上の定番が Celery。

## 役割・なぜ必要か
- メール送信・画像処理・外部API呼び出し・集計などをレスポンスから切り離し、**ユーザーを待たせない**ためにある。
- Django本体は同期リクエスト処理が中心で、Rails の Active Job のような標準キュー機構を持たない。そこで **broker（Redis/RabbitMQ）＋ worker** を別途立てて非同期実行する。
- 構成要素：**broker**（タスクを溜める待ち行列）／**worker**（タスクを取り出して実行するプロセス）／**result backend**（結果の保存先、任意）／**Celery Beat**（定期実行スケジューラ）。

## 基本の書き方（コード）
```python
# proj/celery.py — Celeryアプリ本体（プロジェクト直下）
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "proj.settings")
app = Celery("proj")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()  # 各app の tasks.py を自動収集
```
```python
# proj/__init__.py — Django起動時に Celery を読み込ませる
from .celery import app as celery_app
__all__ = ("celery_app",)
```
```python
# settings.py — broker と result backend（Redisの例）
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/1"
CELERY_TASK_SERIALIZER = "json"
CELERY_TIMEZONE = "Asia/Tokyo"
```
```python
# myapp/tasks.py — タスク定義
from celery import shared_task
from .models import Image

@shared_task(bind=True, max_retries=3, default_retry_delay=10)
def make_thumbnail(self, image_id):   # ★ 引数はモデル本体でなく ID を渡す
    try:
        image = Image.objects.get(pk=image_id)  # worker側で取り直す
        image.generate_thumbnail()
    except Exception as exc:
        raise self.retry(exc=exc)     # 一時障害はリトライ
```
```python
# 呼び出し（View / サービス層から）
from .tasks import make_thumbnail

make_thumbnail.delay(image.id)                       # キューに積む（非同期・通常はこれ）
make_thumbnail.apply_async((image.id,), countdown=60)  # 60秒後に実行（遅延・詳細指定）
```
```bash
# worker と定期実行プロセスの起動（それぞれ常駐させる）
celery -A proj worker -l info        # ワーカー起動（タスクを実行する本体）
celery -A proj beat -l info          # Celery Beat（定期実行スケジューラ）
```
```python
# settings.py — Celery Beat の定期実行（cron相当）
from celery.schedules import crontab
CELERY_BEAT_SCHEDULE = {
    "daily-report": {
        "task": "myapp.tasks.send_daily_report",
        "schedule": crontab(hour=3, minute=0),  # 毎日3:00に実行
    },
}
```

## 実務での使い方・定番パターン
- **重い/遅い処理はリクエスト外へ**：外部API・メール・PDF生成・集計などは `delay()` でworkerに逃がし、Viewは即レスポンスを返す。
- **引数はモデル本体でなくIDを渡す**：モデルインスタンスはJSON化できず、渡せても実行時にデータが古い。`tasks.py` 内で `objects.get(pk=...)` し直す。
- **冪等性を担保**：リトライ・重複enqueueで同じタスクが2回走っても壊れない設計に。処理済みフラグやユニーク制約で二重実行を吸収する。
- **result backend は必要なときだけ**：結果の追跡が要らないタスクは `ignore_result=True` にして backend 負荷を減らす。
- **定期実行は Celery Beat**：cron をアプリ外で管理せず `CELERY_BEAT_SCHEDULE` に集約。スケジュールをDBで管理したい場合は `django-celery-beat` を併用。
- **手動実行・調査**は management command 経由が便利（タスクをコマンドから叩く）。→ [management_commands.md](./management_commands.md)

## ハマりどころ / アンチパターン
- **broker未起動 / 接続先ミス（最頻）**：Redis/RabbitMQ が動いていないと `delay()` がタイムアウト・失敗する。worker・broker・appの設定（`-A proj`）が揃っているか最初に確認。
- **`delay()` 呼んだのに何も起きない**：**workerプロセスを起動し忘れ**ている典型。タスクは積まれただけで、実行する worker がいない。`celery -A proj worker` の常駐が必須。
- **トランザクション内で `delay()`**：コミット前にworkerが走り「まだDBに無いレコード」を引いて失敗する。→ `transaction.on_commit(lambda: task.delay(obj.id))` でコミット後に積む。
- **モデルインスタンスを引数に渡す**：シリアライズ不能 or 鮮度劣化。必ずIDを渡す。
- **冪等でないタスク＋リトライ**：二重課金・重複メールの事故。再実行前提で設計する。
- **デプロイ時の worker 再起動忘れ**：コード更新後も古いタスク定義のworkerが動き続ける。デプロイ手順に**worker再起動**を必ず含める。
- **タスク内で長時間ブロック**：1タスクが重すぎると他が詰まる。粒度を小さく分割し、必要なら専用キュー/worker を分ける。

## 関連
[management_commands.md](./management_commands.md) / [models.md](./models.md) / [settings.md](./settings.md)
