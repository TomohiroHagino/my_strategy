# pytest（FastAPI）

## ひとことで言うと
FastAPIアプリのテストランナー。Starlette 組込の **`TestClient`**（中身は httpx）でサーバを立てずにアプリへHTTPを送り、pytest の `@pytest.fixture` で `TestClient` や依存差し替えを共通化、`@pytest.mark.parametrize` でデータ駆動する。非同期経路を本物で試したいときは `httpx.AsyncClient` ＋ `pytest-asyncio` を使う。

## 役割・なぜ必要か
- API変更のたびに手でcurl/Swaggerを叩くのは非現実的。pytest で**回帰を自動検出**する。
- FastAPIは依存性注入が中心なので、`app.dependency_overrides` での差し替えを fixture にまとめると、DB・認証・設定をテスト用に置換した状態を毎テストで再現できる。
- `parametrize` で「この入力→このステータス/JSON」を一覧で表現でき、実行可能な仕様書になる。

## 基本の書き方（コード）
```bash
pip install pytest httpx           # TestClient は httpx に依存
pip install pytest-asyncio         # 非同期テストを書くなら
```
```python
# test_main.py（同期：TestClient）
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_read_item_ok():
    resp = client.get("/items/foo")          # Act
    assert resp.status_code == 200            # Assert（ステータス）
    assert resp.json() == {"item": "bar"}     # Assert（ボディまで見る）


def test_read_item_not_found():
    resp = client.get("/items/nope")
    assert resp.status_code == 404
    assert resp.json()["detail"] == "Item not found"
```
```python
# conftest.py（fixture で依存差し替え＋クライアントを共通化）
import pytest
from fastapi.testclient import TestClient
from main import app, get_db


def override_get_db():
    db = TestSessionLocal()      # テスト用DB（例: in-memory SQLite）
    try:
        yield db
    finally:
        db.close()


@pytest.fixture
def client():
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:   # with で lifespan(startup/shutdown) も走る
        yield c
    app.dependency_overrides.clear()   # ★ 必ずクリア（次テストへのリーク防止）
```
```python
# parametrize（同じ検証を複数入力で）
import pytest

@pytest.mark.parametrize("payload,status", [
    ({"email": "a@example.com"}, 201),
    ({"email": "bad"}, 422),          # Pydantic バリデーションで 422
    ({}, 422),
])
def test_create_user(client, payload, status):
    resp = client.post("/users", json=payload)
    assert resp.status_code == status
```
```python
# 非同期テスト（httpx.AsyncClient + ASGITransport + pytest-asyncio）
import pytest
import httpx
from main import app


@pytest.mark.asyncio
async def test_async_endpoint():
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as ac:
        resp = await ac.get("/async-items")
    assert resp.status_code == 200
```
```ini
# pytest.ini（asyncio モードを auto にすると @pytest.mark.asyncio 省略可）
[pytest]
asyncio_mode = auto
```

## 実務での使い方・定番パターン
- **`TestClient` を主役に**：実サーバ不要・高速。`client.get/post/put/delete` を撃ち、`resp.status_code` と `resp.json()` の**両方**を検証する。ステータスだけ見て中身を見ないと誤りを取り逃す。
- **fixture で依存差し替えを集約**：`conftest.py` に `client` fixture を置き、`override` → `yield` → `clear()` を1か所に。テストごとに書かなくて済む（→ [dependency_injection.md](./dependency_injection.md)）。
- **`with TestClient(app)` で lifespan を走らせる**：`with` 無しだと startup/shutdown が走らず、起動時初期化に依存するコードが動かない。fixture 内では `with` を使う。
- **`parametrize` でデータ駆動**：正常系・バリデーション 422・404 などをまとめて回す。`ids=[...]` でケース名を付けると失敗が読みやすい。
- **テストDBの方針**：(a) in-memory SQLite で軽く、(b) 本番同等PostgreSQLを別DB/別スキーマで（互換性重視）。各テストでロールバックして独立性を担保（→ [database.md](./database.md)）。
- **非同期は `httpx.AsyncClient` ＋ `ASGITransport`**：`async def` の依存や非同期DBドライバを本物の経路で検証したいとき。`pytest-asyncio`（`asyncio_mode=auto` が楽）か `anyio` を使う。
- **カバレッジ**：`pip install pytest-cov` → `pytest --cov=app --cov-report=term-missing`（目安80%）。

## ハマりどころ
- **`dependency_overrides` のクリア忘れ**：fixture の後処理で `app.dependency_overrides.clear()` を忘れると、差し替えが**次テストへリーク**して順序依存で落ちる。`yield` の後に必ずクリア。
- **テストDB未差し替えで本番/開発DBを汚す**：`get_db` をオーバーライドし忘れると本物のDBに書き込む。fixture でテスト用に必ず置換し、各テストでロールバック/クリーンアップ。
- **`lifespan` が走らない**：`TestClient(app)` を `with` 無しで使うと startup/shutdown が実行されず、起動時に組み立てる接続プール等が `None` のまま落ちる。
- **`asyncio_mode` 未設定**：`async def` テストに `@pytest.mark.asyncio` を付け忘れると **skip 扱いで実行されず**、嘘の緑になる。`asyncio_mode = auto` にすると付け忘れを防げる。
- **同期テストで並行性バグを見逃す**：`TestClient` は多くを検出できるが、並行アクセス特有の不具合は `AsyncClient` での非同期テストが要る。
- **`status_code` だけ見てボディ未検証**：200でも中身が空/誤りのことがある。`resp.json()` の内容まで確認する。
- **過剰なモック**：境界（外部API・DB）だけ差し替え、ロジックは本物を通す。内部実装まで縛ると「もろい」テストになる。

## 関連
[testing.md](./testing.md) / [dependency_injection.md](./dependency_injection.md) / [database.md](./database.md) / [async.md](./async.md)
