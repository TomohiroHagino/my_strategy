# factory_boy（Django 5）

## ひとことで言うと
テストデータを**宣言的に量産する**ライブラリ。`DjangoModelFactory` を継承してモデルごとの「工場」を定義しておけば、`ArticleFactory()` を呼ぶだけで妥当なダミーデータが入ったレコードを作れる。fixtures（固定JSON）より柔軟で、関連オブジェクトも `SubFactory` で芋づる式に生成できる。

## 役割・なぜ必要か
- テストごとに `Model.objects.create(field1=..., field2=..., ...)` を手書きすると、フィールドが増えるたびに全テストを直すことになる。工場に**既定値を集約**すれば、テスト側は「変えたい項目だけ」上書きすればよい。
- 外部キー先（User や Category など）も `SubFactory` で自動生成され、関連の準備が1行で済む。
- `Sequence` / `Faker` で毎回ユニーク・現実的な値を入れられ、ユニーク制約違反や非現実データによる失敗を避けられる。

## 基本の書き方（コード）
```bash
pip install factory-boy   # Faker は依存として一緒に入る
```
```python
# tests/factories.py
import factory
from django.contrib.auth import get_user_model
from myapp.models import Article, Category


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = get_user_model()
    username = factory.Sequence(lambda n: f"user{n}")   # user0, user1, ... 一意
    email = factory.Faker("email")                       # 現実的なダミーメール


class CategoryFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Category
    name = factory.Faker("word")


class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article
    title = factory.Sequence(lambda n: f"title-{n}")
    body = factory.Faker("paragraph")
    author = factory.SubFactory(UserFactory)             # FK先も自動生成
    category = factory.SubFactory(CategoryFactory)
```
```python
# 使う側
a = ArticleFactory()                       # create と同義：DBに保存される
a = ArticleFactory(title="固定タイトル")     # 一部だけ上書き
ArticleFactory.create_batch(3)             # 3件まとめて作成
built = ArticleFactory.build()             # ★ DB保存しない（軽い・FKも保存しない）
```
```python
# pytest と組み合わせる（DBアクセスするので django_db が要る）
import pytest
from tests.factories import ArticleFactory

@pytest.mark.django_db
def test_list_shows_three(client):
    ArticleFactory.create_batch(3)
    resp = client.get("/articles/")
    assert resp.status_code == 200
    assert resp.context["articles"].count() == 3
```
```python
# 状態のバリエーションは Trait や派生クラスで表現
class PublishedArticleFactory(ArticleFactory):
    is_published = True
    published_at = factory.Faker("past_datetime")
```

## 実務での使い方・定番パターン
- **`build()` と `create()` の使い分け**：`create()`（＝呼び出し既定）はDB保存、`build()` はメモリ上だけ。保存不要・FK不要のロジック単体テストは `build()` で速くする。`SubFactory` も `build` 時は保存されない。
- **`Sequence` で一意値**：`username` / `email` など unique 制約のあるフィールドは `factory.Sequence(lambda n: f"user{n}")` で衝突回避。`create_batch` でも安全。
- **`Faker` で現実的な値**：`factory.Faker("email")` / `"name"` / `"paragraph")` / `"past_datetime"`。ロケールは `factory.Faker("name", locale="ja_JP")` で日本語に。
- **`SubFactory` で関連を芋づる生成**：FK先の工場を指定すると親を作るだけで子も揃う。既存の関連を使い回したいときは `ArticleFactory(author=existing_user)` で渡す。
- **`RelatedFactory` / `post_generation`**：逆方向（1対多の「多」側）や M2M を、本体生成後に追加する。タグ付けやコメント生成に使う。
- **`pytest-factoryboy`**：工場を fixture として自動登録できる（`register(ArticleFactory)` で `article` / `article_factory` fixture が生える）。conftest に集約する派の選択肢。
- **fixtures（JSON）と混ぜない**：データソースを factory に一本化する。二重管理は破綻のもと。

## ハマりどころ
- **`build()` なのにFKを保存しようとして落ちる**：`build()` は本体もFK先も保存しない。`build()` したオブジェクトを `.save()` すると、未保存のFK先で `IntegrityError`。保存が要るなら `create()` を使う。
- **`Sequence` のカウンタはプロセス共有**：テスト全体で連番が進む。特定の値を期待するアサートは書かない（`username == "user0"` 前提など）。値は上書き指定する。
- **`Faker` のユニーク非保証**：`factory.Faker("email")` は稀に重複する。unique 制約の列は `Sequence` を使うか `factory.Faker("email")` ではなく `factory.Sequence(lambda n: f"u{n}@example.com")` にする。
- **`SubFactory` の作りすぎで遅い**：`create_batch(100)` が毎回FK先も100個作る。共通の親を1つ作って `author=` で渡し回すと速い。
- **`django_db` 忘れ**：factory は最終的に `objects.create` を呼ぶので、pytest では `@pytest.mark.django_db` が必須（→ [pytest.md](./pytest.md)）。
- **`post_generation` の保存タイミング**：`@factory.post_generation` フックは本体生成後に走る。M2M を足したあと明示 `obj.save()` が要る場合がある（`@factory.post_generation` の `extracted` の扱いに注意）。

## 関連
[pytest.md](./pytest.md) / [testing.md](./testing.md) / [models.md](./models.md)
