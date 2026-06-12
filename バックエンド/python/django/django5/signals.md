# シグナル（signals）（Django 5）

## ひとことで言うと
「保存された」「削除された」などの**イベントが起きたときに、別の関数を自動で呼ぶ**仕組み。送信側（発火元）と受信側（処理）を**疎結合**に繋ぐ通知機構。

## 役割・なぜ必要か
- あるモデルの保存に付随する処理（プロフィール自動作成、キャッシュ破棄、通知送信など）を、**発火元のコードを汚さずに**別の場所へ書ける。
- アプリ間（自分が触れない他アプリのモデル含む）で反応したいときに有効。例：`User` 作成時に自前の `Profile` を自動生成。
- Rails のコールバック（`after_save` 等）に近いが、**処理がモデル本体から物理的に分離される**点が違う。ここが利点でも落とし穴でもある。
- 標準で多くのシグナルが用意されている：`pre_save` / `post_save` / `pre_delete` / `post_delete` / `m2m_changed` / `pre_migrate` / `request_started` など。

## 基本の書き方（コード）
```python
# myapp/signals.py
from django.db.models.signals import post_save, post_delete, m2m_changed
from django.dispatch import receiver
from django.contrib.auth import get_user_model
from .models import Profile, Article

User = get_user_model()


# User が作られたら Profile を自動生成する
@receiver(post_save, sender=User)
def create_profile(sender, instance, created, **kwargs):
    # created=True は「新規作成（INSERT）」のときだけ True
    if created:
        Profile.objects.create(user=instance)


# 記事が削除されたら関連キャッシュを消す
@receiver(post_delete, sender=Article)
def clear_article_cache(sender, instance, **kwargs):
    from django.core.cache import cache
    cache.delete(f"article:{instance.pk}")


# ManyToMany（tags）の付け外しに反応する
@receiver(m2m_changed, sender=Article.tags.through)
def on_tags_changed(sender, instance, action, pk_set, **kwargs):
    # action は "pre_add"/"post_add"/"pre_remove"/"post_remove"/"post_clear"
    if action == "post_add":
        print(f"{instance} にタグ {pk_set} を追加")
```

```python
# myapp/apps.py … ready() で接続するのが正解
from django.apps import AppConfig


class MyappConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "myapp"

    def ready(self):
        # import するだけで @receiver が登録される
        from . import signals  # noqa: F401
```

## 実務での使い方・定番パターン
- **接続は `apps.py` の `ready()` で行う**のが定石。`models.py` 末尾に書くとアプリ初期化順で二重接続・未接続が起きやすい。`AppConfig` は `apps.py` に置けば `INSTALLED_APPS` で自動検出される（`default_app_config` は不要）。
- **`created` 引数で「新規 / 更新」を分ける**。`post_save` は INSERT でも UPDATE でも飛ぶので、新規時だけの処理は `if created:` で囲む。
- **`dispatch_uid` で二重登録を防ぐ**。`ready()` が複数回呼ばれても受信が重複しないよう一意キーを付ける。
  ```python
  @receiver(post_save, sender=User, dispatch_uid="create_profile_once")
  def create_profile(sender, instance, created, **kwargs):
      ...
  ```
- **`update_fields` で対象を絞る**。特定フィールド更新時のみ反応させたいとき。
  ```python
  @receiver(post_save, sender=Article)
  def on_status_change(sender, instance, update_fields=None, **kwargs):
      if update_fields and "status" in update_fields:
          ...  # status を更新した保存のときだけ
  ```
- **カスタムシグナルも作れる**。自前イベントを `Signal()` で定義し `.send()` で発火。

## ハマりどころ / アンチパターン
- **暗黙の副作用で追跡困難**。保存しただけで遠くの関数が走るため、「なぜこのレコードが作られた？」が読めなくなる。**コードを追って原因を特定しにくい**のが最大の弱点。チームで使うなら「どのシグナルがどこに繋がるか」を一覧化しておく。
- **過剰使用は避ける**。**同じアプリ内・自分が制御できる処理なら、シグナルより明示的なメソッド呼び出しの方が読みやすい**ことが多い。`signals` が真価を発揮するのは「他アプリのモデルに反応したい」など分離が必要な場面に限る。→ [models.md](./models.md)
- **テスト中の意図せぬ発火**。フィクスチャ作成やファクトリで `post_save` が走り、テストが汚れる/遅くなる。対策＝テストで一時的に切断する。
  ```python
  from unittest.mock import patch
  # 例：受信関数をモックして発火しても何もしない
  with patch("myapp.signals.create_profile"):
      User.objects.create(username="x")
  # あるいは signal.disconnect(receiver, sender=...) で外す
  ```
- **`bulk_create` / `update()` / `delete()`(QuerySet) では発火しない**。`post_save` は `save()` を、`post_delete` はインスタンスの `delete()` を経由したときに飛ぶ。一括系はシグナルを通らないので、一括処理に依存ロジックを乗せている場合は要注意。
- **`pre_save` での重い処理は保存をブロックする**。同期実行なのでメール送信・外部API等はトランザクションを長く保持する。重い処理は Celery など非同期へ逃がす。→ [celery.md](./celery.md)
- **無限ループ**。`post_save` の中で同じインスタンスを `save()` するとシグナルが再発火し続ける。更新は `update_fields` 指定や `queryset.update()` で回避する。

## 関連: [models.md](./models.md) / [celery.md](./celery.md)
