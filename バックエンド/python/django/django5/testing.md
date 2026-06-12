# テスト（Testing）（Django 5）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。Django標準は **`django.test.TestCase`**（unittestベース）。実務では **pytest-django** が定番で、`factory_boy` でテストデータを作る。テストは**専用のテスト用DB**で走り、各テストはトランザクションでロールバックされる。

## 役割・なぜ必要か
- 変更のたびに手で全画面を確認するのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- 「この仕様で動く」という実行可能な仕様書になり、リファクタの安全網になる。
- Django標準のテストクライアントで、実際にURLを叩いてView・テンプレート・ORMの結合まで一気に検証できる。

## 基本の書き方（コード）
```python
# myapp/tests.py（Django標準 TestCase）
from django.test import TestCase
from django.urls import reverse
from myapp.models import Article

class ArticleViewTest(TestCase):
    def setUp(self):
        # 各テストの前に毎回呼ばれる。テストDBは前テストの状態を持ち越さない
        self.article = Article.objects.create(title="hello", body="本文")

    def test_detail_returns_200(self):
        url = reverse("article_detail", args=[self.article.pk])
        response = self.client.get(url)         # テストクライアントでGET
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "hello")  # bodyに文字列を含むか

    def test_create_via_post(self):
        response = self.client.post(reverse("article_create"),
                                    {"title": "new", "body": "x"})
        self.assertEqual(response.status_code, 302)   # 作成後リダイレクト
        self.assertTrue(Article.objects.filter(title="new").exists())
```

```python
# pytest-django（実務人気）: tests/test_article.py
import pytest
from django.urls import reverse
from myapp.models import Article

@pytest.mark.django_db          # DBアクセスするテストには必須のマーカー
def test_detail(client):        # client フィクスチャがテストクライアント
    a = Article.objects.create(title="hello", body="本文")
    resp = client.get(reverse("article_detail", args=[a.pk]))
    assert resp.status_code == 200
    assert "hello" in resp.content.decode()
```

```python
# factory_boy でテストデータを宣言的に生成: tests/factories.py
import factory
from myapp.models import Article

class ArticleFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Article
    title = factory.Sequence(lambda n: f"title-{n}")
    body = "本文"

# 使う側: a = ArticleFactory()  /  ArticleFactory.create_batch(3)
```

## 実務での使い方・定番パターン
- **pytest-django が主流**：`pytest.ini` / `pyproject.toml` に `DJANGO_SETTINGS_MODULE` を指定。`pytest` 一発で実行でき、`assert` がそのまま書けて読みやすい。
- **テストクライアント**：`self.client.get/post(...)` でHTTPを模擬し、`assertEqual(status_code)`・`assertContains`・`assertRedirects`・`assertTemplateUsed` で検証。ログインは `self.client.force_login(user)`。
- **`setUp` vs `setUpTestData`**：`setUpTestData`（クラス単位で1回）は速い。各テストで書き換えないデータはこちらに。
- **factory_boy** でデータ生成を宣言的に。fixtures（JSON）より柔軟で、関連オブジェクトも `SubFactory` で芋づる生成できる。
- **`TestCase` はトランザクションでロールバック**：各テスト後にDBが巻き戻るので、テスト同士が独立する。トランザクション挙動そのものを試すなら `TransactionTestCase`。
- **`--keepdb`**：`pytest --reuse-db` / `manage.py test --keepdb` でテストDBを再利用し、毎回のマイグレーション適用を省いて高速化。
- **カバレッジ**：`coverage run -m pytest` → `coverage report`（目安80%）。重要フローを優先。

## ハマりどころ / アンチパターン
- **テスト用DBの取り違え**：テストは本番/開発DBではなく**自動生成のテストDB**で走る。`@pytest.mark.django_db` を付け忘れると `RuntimeError: Database access not allowed`。
- **外部依存をモックしない**：外部API・メール送信・課金・時刻などはモック必須。メールは `django.core.mail.outbox` で検証、時刻は `freezegun`、HTTPは `responses`/`unittest.mock` で差し替える。実通信は遅く不安定（flaky）。
- **テスト間の状態リーク**：モジュールグローバル・キャッシュ（`cache`）・ファイル生成が残り、順序依存で落ちる。`setUp`/`tearDown` で確実に初期化、キャッシュは `LocMemCache` を使い分離。
- **過剰なモック**：実装内部を縛りすぎると、リファクタで赤くなる「もろい」テストに。境界（外部I/O）だけモックする。
- **fixtures と factory の混在**：データソースが二重化し管理破綻。実務はfactory寄りに統一。
- **`create` の乱用で遅い**：DB保存が不要なら `build`（`factory.build`）で軽く。
- カバレッジ数値だけ追って**重要フローの結合テストが無い**のは本末転倒。

## 関連
[models.md](./models.md) / [orm_queries.md](./orm_queries.md) / [views.md](./views.md) / [auth.md](./auth.md)
