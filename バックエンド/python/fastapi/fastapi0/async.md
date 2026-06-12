# `async`/`await`（いつ非同期にすべきか）

## ひとことで言うと
FastAPIはパスオペレーションを **`async def` でも普通の `def` でも書ける**。`async def` は非同期（`await` でI/O待ちの間に他の処理を進める）、`def` は同期だがFastAPIが**スレッドプールで実行**してくれる。どちらを選ぶかが性能を左右する。

## 役割・なぜ必要か
- FastAPIはASGI（非同期サーバ：Uvicorn等）上で動く。1つの**イベントループ**が多数のリクエストを捌く。`await` で待つ間にループは別リクエストを処理でき、これが高スループットの源。
- だが**ループは1本**。`async def` の中で「待つ」べき箇所をブロッキング（同期で固まる）処理にすると、その間ループ全体が止まり、**全リクエストが詰まる**。これがasyncの最大の落とし穴。
- 逆に普通の `def` で書いたルートは、FastAPIが自動で**別スレッド（スレッドプール）**に逃がして実行する。だからブロッキングな同期ライブラリ（多くのDBドライバ・`requests` 等）は `def` で書けば安全。
- 判断軸はシンプル：**待ち時間に `await` できる非同期ライブラリがあるか**。あれば `async def`、無ければ `def`。

## 基本の書き方（コード）
```python
import asyncio, time
import httpx
from fastapi import FastAPI

app = FastAPI()

# (1) I/Oバウンド × 非同期ライブラリ → async def ＋ await（理想形）
@app.get("/weather")
async def weather():
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://api.example.com/weather")  # 待つ間に他を処理
    return resp.json()

# (2) 並行実行：複数の外部APIを同時に叩く（直列なら遅い）
@app.get("/dashboard")
async def dashboard():
    async with httpx.AsyncClient() as client:
        a, b = await asyncio.gather(           # 同時に待つ
            client.get("https://api.example.com/orders"),
            client.get("https://api.example.com/users"),
        )
    return {"orders": a.json(), "users": b.json()}

# (3) 同期ライブラリ（ブロッキング）→ def にする（FastAPIがスレッドへ逃がす）
@app.get("/legacy")
def legacy_call():
    data = blocking_db_query()   # 同期DBドライバ等。def なら安全
    return data
```

```python
# (4) アンチパターン：async def の中でブロッキング呼び出し（最重要）
@app.get("/bad")
async def bad():
    time.sleep(3)            # NG: イベントループ全体が3秒止まる→全員巻き添え
    return {"ok": True}

# 正しくは：非同期で待つ、または同期処理をスレッドへ逃がす
@app.get("/good")
async def good():
    await asyncio.sleep(3)                       # OK: ループを塞がない
    # CPU重い/同期処理はスレッドへ：
    # result = await asyncio.to_thread(cpu_heavy_func, arg)
    return {"ok": True}
```

## 実務での使い方・定番パターン
- **I/Oバウンド（DB・外部API・ファイル）で非同期対応ライブラリがある** → `async def` ＋ `await`。HTTPは `httpx.AsyncClient`、DBは `asyncpg` / SQLAlchemy 2.0 async など。→ [database.md](./database.md)
- **同期ライブラリしか無い（`requests`・同期DBドライバ・既存資産）** → 素直に `def` で書く。FastAPIがスレッドプールで回すのでループは塞がない。無理にラップしない。
- **複数の独立したI/Oは並行化**：`await asyncio.gather(...)` で同時実行。直列で `await` を並べると待ち時間が足し算になる。
- **CPUバウンド（画像処理・重い計算・ML推論）**：`async` にしてもイベントループを占有して逆効果。`await asyncio.to_thread(...)` でスレッドへ、本当に重ければ別プロセス/ワーカー（Celery等）へ。→ [background_tasks.md](./background_tasks.md)
- **async内で同期処理をどうしても呼ぶ**なら `await asyncio.to_thread(func, *args)`（または `run_in_executor`）で逃がす。
- **依存関数も `async def` 可**：`yield` 依存も非同期で書ける。中身がブロッキングなら同期 `def` 依存にする。→ [dependency_injection.md](./dependency_injection.md)

## ハマりどころ / アンチパターン
- **`async def` 内のブロッキング呼び出し**（最頻出・最重要）：`time.sleep()`・同期DBクエリ・`requests.get()` 等を `async def` で呼ぶと**イベントループが固まり全リクエストが詰まる**。`await` 可能な非同期版を使うか、関数を `def` にする。
- **「とりあえず全部 `async def`」**：中で `await` を一切使わない `async def` は、スレッドにも逃げず利点ゼロ。むしろブロッキングを混ぜると害。`await` するものが無いなら `def` にする。
- **`async def` の中で同期DBドライバ**：`psycopg2` 等は同期。async対応の `asyncpg` を使うか、ルートを `def` にする。中途半端が一番危険。
- **`await` の付け忘れ**：`await` し忘れるとコルーチンオブジェクトが返るだけで実行されない（警告は出るが結果は空）。
- **CPUバウンドをasyncで解決しようとする**：非同期は「待ち」を効率化するもので、計算は速くならない。CPU処理はスレッド/プロセス/外部ワーカーへ。
- **ブロッキングなライブラリを `asyncio` で無理に包む**：かえって複雑化。同期は `def` ＋スレッドプール任せが素直。
- **スレッドプールの枯渇**：`def` ルートが大量＆長時間だとスレッドが尽きる。本質的に非同期化すべきか、ワーカー数調整を検討。

## 関連
[dependency_injection.md](./dependency_injection.md) / [database.md](./database.md) / [background_tasks.md](./background_tasks.md)
