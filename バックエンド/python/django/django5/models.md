# モデル（Model / ORM）（Django 5）

## ひとことで言うと
DBのテーブル1つに対応するPythonクラスで、**データの形（フィールド定義）＋そのデータに関する業務ロジックの置き場**。`django.db.models.Model` を継承する。Railsの `ApplicationRecord` 相当だが、Djangoは**スキーマ定義もモデルクラスに書く**（マイグレーションがそこから生成される）のが特徴。

## 役割・なぜ必要か
- テーブル `app_article` ↔ クラス `Article`、1行 ↔ 1インスタンス、という対応で「SQLを直接書かずにPythonでDBを扱う」ためにある。
- MTV（Model-Template-View）の中で「**何を・どう扱うか（業務ルール）**」を担う中心。Viewやテンプレートに業務ルールを散らさず、ここへ集約することで再利用・テストがしやすくなる。
- **フィールド定義 → `makemigrations` → `migrate`** という流れでDBスキーマと同期する。モデルが「正」、DBはそれに追従する。

## 基本の書き方（コード）
```python
# app/models.py
from django.db import models
from django.conf import settings


class Category(models.Model):
    name = models.CharField(max_length=50, unique=True)

    def __str__(self):  # 管理画面や print 表示で使われる。必ず定義する
        return self.name


class Article(models.Model):
    class Status(models.TextChoices):       # 状態管理は choices で
        DRAFT = "draft", "下書き"
        PUBLISHED = "published", "公開"

    title = models.CharField(max_length=200)                    # 短い文字列（VARCHAR）
    body = models.TextField(blank=True)                         # 長文（TEXT）。空文字許容
    view_count = models.IntegerField(default=0)                 # 整数
    status = models.CharField(max_length=10, choices=Status.choices, default=Status.DRAFT)
    published_at = models.DateTimeField(null=True, blank=True)  # 日時。NULL許容
    created_at = models.DateTimeField(auto_now_add=True)        # 作成時に自動セット
    updated_at = models.DateTimeField(auto_now=True)           # 保存ごとに自動更新

    # 関連: 多対1（記事は1ユーザーに属する）。on_delete は必須引数
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,        # 親（ユーザー）削除で記事も削除
        related_name="articles",         # user.articles.all() で逆参照
    )
    # 関連: 多対多（記事は複数カテゴリを持てる）
    categories = models.ManyToManyField(Category, related_name="articles", blank=True)

    class Meta:
        ordering = ["-published_at"]                 # 既定の並び順（新しい順）
        indexes = [models.Index(fields=["status", "published_at"])]
        constraints = [
            models.UniqueConstraint(fields=["author", "title"], name="uniq_author_title"),
        ]

    def __str__(self):
        return self.title

    def is_published(self):  # ドメインロジックはここ（fat controller を避ける）
        return self.status == self.Status.PUBLISHED
```

## 実務での使い方・定番パターン
- **フィールド型の使い分け**: `CharField`(要 `max_length`)/`TextField`(長文)/`IntegerField`・`PositiveIntegerField`/`BooleanField`/`DateTimeField`/`DecimalField`(金額)/`EmailField`/`SlugField` など。用途に合った型を選ぶとバリデーションとDB型が適切になる。
- **関連の3種**:
  - `ForeignKey`（多対1）… 最頻。記事→著者。
  - `ManyToManyField`（多対多）… 記事↔タグ。中間テーブルは自動生成（`through=` で明示も可）。
  - `OneToOneField`（1対1）… ユーザー↔プロフィール拡張。
- **`on_delete` の選択**（`ForeignKey`/`OneToOneField`は必須）:
  - `CASCADE`（親削除で子も削除）/ `PROTECT`（子が居れば親削除を例外で防ぐ）/ `SET_NULL`（`null=True`必須）/ `SET_DEFAULT` / `RESTRICT`。
- **`related_name`** を付けると逆参照が読みやすい（`user.articles.all()`）。付けないと `article_set` になる。
- **`class Meta`** で `ordering` / `db_table` / `indexes` / `constraints`（DB制約）/ `verbose_name` を設定。一意制約は `UniqueConstraint` で張るのが堅い（アプリ層バリデーションと二重に）。
- **`__str__`** は必ず定義する。管理画面・shell・ログでの表示がこれで決まる。
- **Django 5 の新機能**: `GeneratedField`（DB側で計算した値を持つ列）、`db_default`（DBレベルのデフォルト値）。
- スキーマを変えたら必ず `python manage.py makemigrations` → `migrate`。→ [migrations.md](./migrations.md)

```bash
python manage.py makemigrations app   # モデル変更を検出してマイグレーション生成
python manage.py migrate               # DBへ適用
python manage.py shell                  # 5.2 はモデル自動importでそのまま Article が使える
```

## ハマりどころ / アンチパターン
- **`__str__` 未定義**: 管理画面が `Article object (1)` のように表示され何の行か分からない。最初に定義する。
- **`null` と `blank` の混同**（重要）: `null=True` は**DBにNULLを許す**（DB制約）、`blank=True` は**フォーム/バリデーションで空欄を許す**（アプリ層）。文字列系（`CharField`/`TextField`）はNULLと空文字が二重に空を表せて事故るので、原則 `blank=True` のみ（`null=True` は付けない）。`DateTimeField` などNULLが必要な型は両方付ける。
- **`on_delete` 付け忘れ**: `ForeignKey`/`OneToOneField` で `on_delete` は必須。書かないと `TypeError`。意味を理解せず `CASCADE` を機械的に付けると、ユーザー削除で大量データが消える事故に。重要データは `PROTECT` を検討。
- **fat model（太りすぎ）**: 複数モデルにまたがる手続きや外部APIなどの副作用をモデルに詰め込むと肥大化。サービス関数や `managers`/`querysets` 分離で整理する。一方で、薄すぎてViewにロジックが漏れるのも避ける（fat controller）。
- **マイグレーション忘れ**: モデルを変えたのに `makemigrations` し忘れ → コードとDBスキーマが不整合に。`makemigrations --check` をCIに入れると検出できる。
- **`auto_now` / `auto_now_add` の上書き不可**: これらは自動管理で、手動代入しても保存時に上書きされる。任意セットしたいなら `default=timezone.now` を使う。

## 関連
[orm_queries.md](./orm_queries.md) / [migrations.md](./migrations.md) / [admin.md](./admin.md) / [forms.md](./forms.md)
