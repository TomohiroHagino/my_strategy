# マイグレーション（Migrations）（Django 5）

## ひとことで言うと
モデル（`models.py`）の変更を**Pythonファイルとして記録し、DBスキーマに反映する仕組み**。「モデルを変える → `makemigrations` で差分ファイル生成 → `migrate` でDBに適用」という流れで、スキーマ変更をバージョン管理する。Railsのマイグレーション相当だが、Djangoは**モデル定義から差分を自動生成する**点が違う。

## 役割・なぜ必要か
- スキーマ変更を手書きSQLでなくコードで管理し、開発・ステージング・本番で**同じ順序で再現**できるようにするためにある。
- 各マイグレーションは依存関係（`dependencies`）で順序付けられ、適用済みかどうかは `django_migrations` テーブルで管理される。チーム開発で「誰がどの変更をいつ入れたか」を追える。

## 基本の書き方（コード）
```bash
# モデルを変更したら、差分を検出してマイグレーションファイルを生成
python manage.py makemigrations app

# 生成された変更をDBに適用
python manage.py migrate

# 適用状況を一覧（[X]=適用済み）
python manage.py showmigrations app

# あるマイグレーションが発行するSQLを確認（適用はしない）
python manage.py sqlmigrate app 0002

# 特定マイグレーションまで戻す（0001 の状態へロールバック）
python manage.py migrate app 0001
```

生成されるファイル例（`app/migrations/0002_article_status.py`）:
```python
from django.db import migrations, models


class Migration(migrations.Migration):
    dependencies = [("app", "0001_initial")]   # この前に必要なマイグレーション

    operations = [
        migrations.AddField(
            model_name="article",
            name="status",
            field=models.CharField(default="draft", max_length=10),
        ),
    ]
```

## 実務での使い方・定番パターン
- **基本サイクル**: `models.py` 編集 → `makemigrations` → 生成ファイルを目視確認 → `migrate`。生成ファイルは**必ずコミット**する（DBの正史になる）。
- **データマイグレーション**（既存データの変換）: スキーマだけでなくデータも移す場合は `RunPython` を使う。逆操作も書くとロールバック可能になる。
  ```python
  from django.db import migrations

  def set_default_status(apps, schema_editor):
      Article = apps.get_model("app", "Article")   # 過去時点のモデルを取得（重要）
      Article.objects.filter(status="").update(status="draft")

  def reverse(apps, schema_editor):
      pass  # 逆操作（不要なら no-op、不可逆なら RunPython.noop）

  class Migration(migrations.Migration):
      dependencies = [("app", "0002_article_status")]
      operations = [migrations.RunPython(set_default_status, reverse)]
  ```
  - `RunPython` 内では `apps.get_model()` で**そのマイグレーション時点のモデル**を取る。`from app.models import Article` を直接importすると将来モデルが変わって壊れる。
- **空マイグレーションを作る**: `python manage.py makemigrations --empty app` でデータ移行用の雛形を生成。
- **squash（統合）**: マイグレーションが増えすぎたら `python manage.py squashmigrations app 0001 0050` で1ファイルに圧縮し、起動・テストを高速化。
- **CIで差分検知**: `python manage.py makemigrations --check --dry-run` を入れ、`makemigrations` 忘れを検出する。
- **後方互換な順序**: カラム追加は安全。削除・リネームは「追加 → 両対応コードをデプロイ → 後で削除」の段階移行が安全。

## ハマりどころ / アンチパターン
- **`makemigrations` 忘れ**: モデルを変えたのに生成し忘れ → コードとDBスキーマが不整合になり、本番で `column does not exist` 等のエラー。`--check` をCIに入れて防ぐ。
- **マイグレーション競合（conflict）**: 複数人が同じ親（例 `0005`）から別々に `0006_a` / `0006_b` を作るとブランチが分岐し、`migrate` 時に `Conflicting migrations` エラー。`python manage.py makemigrations --merge` でマージマイグレーションを作って解消する。
- **不可逆操作**: カラム削除・型変更・データ削除は元に戻せない。`RunPython` には逆関数（最低でも `RunPython.noop`）を書き、本番適用前にステージングでロールバックも試す。
- **本番の長時間ロック**: 大きなテーブルへの `AddField`（NOT NULL + default）やインデックス作成はテーブルをロックし、その間アクセスが詰まる。PostgreSQLなら `AddIndexConcurrently`（非ブロッキング索引作成）や、まず `null=True` で追加→バックフィル→制約付与、の分割が定石。
- **生成ファイルを手で雑に書き換える**: 依存関係や状態がずれて壊れやすい。基本は自動生成に任せ、データ移行など必要な部分だけ追記する。
- **`migrate` をアプリ起動と同時に毎回走らせる**: 複数インスタンス同時起動でレースになる。デプロイ手順で**1回だけ**明示的に実行する。

## 関連
[models.md](./models.md) / [orm_queries.md](./orm_queries.md) / [settings.md](./settings.md)
