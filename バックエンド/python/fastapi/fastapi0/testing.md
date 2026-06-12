# テスト（Testing）（FastAPI）

## ひとことで言うと
FastAPIアプリの振る舞いを**自動で検証するコード**。標準は **`TestClient`**（内部は httpx ベース）で、サーバを立てずにアプリへHTTPリクエストを送って検証する。`pytest` と組み合わせ、**`app.dependency_overrides` でDBや認証などの依存を差し替える**のが肝。

## 役割・なぜ必要か
- API変更のたびに手でcurl/Swaggerを叩くのは非現実的。**回帰（デグレ）を自動検出**するために要る。
- FastAPIは依存性注入が中心なので、**`dependency_overrides` で外部依存（本物のDB・外部API）をテスト用に差し替え**られ、速く安定したテストが書ける。
- 「この入力でこのステータス/JSONが返る」という実行可能な仕様書になり、リファクタの安全網になる。

## 基本の書き方（コード）
```python
# test_main.py（同期: TestClient）
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)


def test_read_item_ok():
    # Arrange / Act
    resp = client.get("/items/foo")
    # Assert
    assert resp.status_code == 200
    assert resp.json() == {"item": "bar"}


def test_read_item_not_found():
    resp = client.get("/items/nope")
    assert resp.status_code == 404
    assert resp.json()["detail"] == "Item not found"
```

```python
# 依存の差し替え（DBセッションをテスト用に置換）
from main import app, get_db


def override_get_db():
    db = TestSessionLocal()      # テスト用DB（例: in-memory SQLite）
    try:
        yield db
    finally:
        db.close()


# get_db の代わりに override_get_db を使わせる
app.dependency_overrides[get_db] = override_get_db


def test_create_user():
    resp = client.post("/users", json={"email": "a@example.com"})
    assert resp.status_code == 201
    # テスト後に元へ戻す（漏れ防止）
    app.dependency_overrides.clear()
```

```python
# 非同期テスト（httpx.AsyncClient + ASGITransport）
import pytest
import httpx
from main import app


@pytest.mark.anyio
async def test_async_endpoint():
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as ac:
        resp = await ac.get("/async-items")
    assert resp.status_code == 200
```

```python
# conftest.py（fixtureで依存差し替えを共通化）
import pytest
from fastapi.testclient import TestClient
from main import app, get_db


@pytest.fixture
def client():
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:   # with でlifespan(startup/shutdown)も走る
        yield c
    app.dependency_overrides.clear()   # 必ずクリア
```

## 実務での使い方・定番パターン
- **`TestClient` を主役に**：実サーバ不要・高速。`client.get/post/put/delete` でHTTPを送り、`resp.status_code` と `resp.json()` を検証する。
- **`app.dependency_overrides` で差し替え**：DBセッション・認証ユーザ・設定（`get_settings`）などを丸ごとテスト用に置換。本物の外部依存に触れず速く安定する（→ [dependency_injection.md](./dependency_injection.md)）。
- **テストDBの方針**：(a) in-memory SQLite で軽く回す、(b) 本番同等のPostgreSQLを別DB/別スキーマで用意（互換性重視）。トランザクションを各テストでロールバックして独立性を担保（→ [database.md](./database.md)）。
- **fixtureで前後処理を共通化**：`conftest.py` に `client` fixtureを置き、差し替え→`yield`→`clear()` を1か所に。`with TestClient(app)` を使うと **lifespan（startup/shutdown）も実行**される。
- **非同期は `httpx.AsyncClient` + `ASGITransport`**：`async def` の依存やDBドライバを本物の非同期経路で検証したいとき。`pytest-asyncio` か `anyio` を使う。
- **AAA構造**：Arrange（データ準備）→ Act（リクエスト）→ Assert（検証）で読みやすく。カバレッジは `pytest --cov` で計測し80%目安。

## ハマりどころ / アンチパターン
- **テストDBを用意せず本番/開発DBを汚す**：`dependency_overrides` を忘れると**本物のDBに書き込む**。必ずテスト用DBへ差し替え、各テストでロールバック/クリーンアップ。
- **依存オーバーライド忘れ・クリア忘れ**：(a) 差し替えを忘れて外部依存に触れる、(b) `clear()` 忘れで**次のテストへ差し替えがリーク**し順序依存で落ちる。fixtureの後処理で必ず `app.dependency_overrides.clear()`。
- **テスト間の状態リーク**：モジュールレベルの `app.dependency_overrides[...] = ...` は他テストにも効く。fixtureでスコープを閉じる。
- **`lifespan` が走らないことに気付かない**：`TestClient(app)` を `with` 無しで使うと startup/shutdown が走らず、起動時初期化に依存するコードが動かない。`with TestClient(app) as client:` を使う。
- **同期テストで非同期だけの不具合を見逃す**：`TestClient` は内部でイベントループを回すため多くは検出できるが、並行性バグは `AsyncClient` での非同期テストが要る。
- **過剰なモック**：実装内部を縛りすぎるとリファクタで赤くなる「もろい」テストに。**境界（外部API・DB）だけ差し替え**、ロジックは本物を通す。
- **`status_code` だけ見てボディ未検証**：200でも中身が空/誤りのことがある。`resp.json()` の内容まで検証する。

## 関連
[dependency_injection.md](./dependency_injection.md) / [database.md](./database.md) / [error_handling.md](./error_handling.md)
