# pytest / pytest-django（Django 5）

## ひとことで言うと
Python標準の `unittest` の代わりに使う**テストランナー**が pytest。`assert` がそのまま書け、`pytest-django` プラグインを足すと Django の設定読み込み・テストDB作成・`client` フィクスチャが使えるようになる。DBアクセスするテストには `@pytest.mark.django_db` を付けるのが必須。

## 役割・なぜ必要か
- Django標準の `TestCase`（クラス継承＋`self.assert...`）より、**関数＋素の `assert` で短く書ける**。失敗時の差分表示も pytest が自動で見やすく出す。
- `pytest-django` が `DJANGO_SETTINGS_MODULE` を読み、テスト用DBを自動生成・各テストでロールバックしてくれる。`client` フィクスチャでHTTPも叩ける。
- `fixture` / `parametrize` / `conftest.py` で前処理とデータ駆動を整理でき、テストが増えても破綻しにくい。

## 基本の書き方（コード）
```bash
pip install pytest pytest-django
```
```ini
# pytest.ini（or pyproject.toml の [tool.pytest.ini_options]）
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py *_tests.py
```
```python
# tests/test_article.py
import pytest
from django.urls import reverse
from myapp.models import Article


@pytest.mark.django_db                 # DBアクセスするテストには必須
def test_detail_returns_200(client):   # client フィクスチャ＝テストクライアント
    a = Article.objects.create(title="hello", body="本文")
    resp = client.get(reverse("article_detail", args=[a.pk]))
    assert resp.status_code == 200
    assert "hello" in resp.content.decode()


@pytest.mark.django_db
def test_create_via_post(client):
    resp = client.post(reverse("article_create"), {"title": "new", "body": "x"})
    assert resp.status_code == 302                      # 作成後リダイレクト
    assert Article.objects.filter(title="new").exists()
```
```python
# parametrize（同じ検証を複数入力で回す＝データ駆動）
@pytest.mark.parametrize("title,expected_status", [
    ("valid", 302),
    ("", 200),          # 空はバリデーションで再表示
])
@pytest.mark.django_db
def test_create_validation(client, title, expected_status):
    resp = client.post(reverse("article_create"), {"title": title, "body": "x"})
    assert resp.status_code == expected_status
```
```python
# conftest.py（同ディレクトリ配下で共有する fixture を置く）
import pytest
from django.contrib.auth import get_user_model


@pytest.fixture
def user(db):                          # db フィクスチャ＝django_db 相当
    return get_user_model().objects.create_user(username="taro", password="pw")


@pytest.fixture
def auth_client(client, user):         # フィクスチャは他のフィクスチャを引数に取れる
    client.force_login(user)
    return client
```
```python
# 上の fixture を使う側（引数名で注入される）
def test_dashboard_requires_login(auth_client):
    resp = auth_client.get("/dashboard/")
    assert resp.status_code == 200
```

## 実務での使い方・定番パターン
- **`@pytest.mark.django_db` を DBテストに必ず付ける**：付けないと `RuntimeError: Database access not allowed`。読み取りだけでも付ける。クラス全体に付けるなら `pytestmark = pytest.mark.django_db` をモジュール先頭に書く。
- **`client` / `admin_client` / `rf` フィクスチャ**：`client` はテストクライアント、`admin_client` は管理者ログイン済み、`rf`（RequestFactory）は View を直接呼ぶとき。ログインは `client.force_login(user)`。
- **`conftest.py` に共通 fixture を集約**：プロジェクト直下の `conftest.py` は全テストで、サブディレクトリのものはその配下で有効。`scope="session"` で重い準備を1回だけにできる。
- **`parametrize` でデータ駆動**：正常系・異常系・境界値をまとめて回す。`ids=[...]` でケース名を付けると失敗箇所が読みやすい。
- **`--reuse-db` で高速化**：`pytest --reuse-db` でテストDBを再利用し、毎回のマイグレーション適用を省く。スキーマ変更時だけ `--create-db`。
- **マーカーで絞り込み**：`@pytest.mark.slow` を付けて `pytest -m "not slow"`。`pytest -k "detail"` は名前で絞る。
- **カバレッジ**：`pip install pytest-cov` → `pytest --cov=myapp --cov-report=term-missing`（目安80%）。重要フローを優先。
- **`factory_boy` と組み合わせる**：データ生成は fixture 内で `ArticleFactory()` を呼ぶのが定番（→ [factory_boy.md](./factory_boy.md)）。

## ハマりどころ
- **`django_db` 付け忘れ**：DBに触れた瞬間 `RuntimeError`。逆に不要なテストに付けるとDBセットアップ分だけ遅くなる。
- **`DJANGO_SETTINGS_MODULE` 未設定**：`pytest.ini` / `pyproject.toml` に書き忘れると `ImproperlyConfigured` で起動しない。環境変数で渡してもよいが設定ファイルに固定が安全。
- **`db` と `django_db` の混同**：fixture の引数では `db`（`@pytest.fixture` 内で使う組込フィクスチャ）、テスト関数のデコレータでは `@pytest.mark.django_db`。役割は同じDB許可だが書く場所で名前が違う。
- **fixture のスコープ取り違え**：`scope="session"` の fixture でDBを書き換えると、他テストへ状態がリークする。書き換える可能性があるものは関数スコープ（既定）に。
- **`parametrize` と `django_db` の順序**：デコレータは下から上に適用される。`@parametrize` を外側、`@django_db` を内側に重ねるのが無難（どちらの順でも動くが意図を揃える）。
- **トランザクション挙動を試したいとき**：`TestCase` 相当はトランザクションでロールバックするため、`transaction.on_commit` 等はそのままでは発火しない。`@pytest.mark.django_db(transaction=True)` を使う（遅い）。
- **外部依存をモックしない**：メールは `django.core.mail.outbox`、時刻は `freezegun`、HTTPは `responses` で差し替える。実通信は遅く不安定。

## 関連
[testing.md](./testing.md) / [factory_boy.md](./factory_boy.md) / [orm_queries.md](./orm_queries.md)
