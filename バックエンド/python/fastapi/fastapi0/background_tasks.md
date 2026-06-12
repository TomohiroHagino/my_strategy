# バックグラウンドタスク（FastAPI）

## ひとことで言うと
**レスポンスを返した「後」に、同じプロセス内で実行する軽い後処理**。メール送信・通知・簡単なログ書き込みなどを「クライアントを待たせずに」やるための仕組み。Starlette 由来の `BackgroundTasks` を**パスオペレーション関数の引数で受け取り**、`add_task` で積む。

## 役割・なぜ必要か
- 「処理は必要だが、**結果をレスポンスに含める必要がない**」ものを切り離し、API の応答を速くする。
- 例：登録完了レスポンスは即返しつつ、**ウェルカムメールはその後ろで送る**。ユーザーはメール送信完了を待たない。
- ただし FastAPI は**ジョブキューを持たない**。`BackgroundTasks` は**同一プロセス・同一イベントループ内**で動く軽量な仕組みであり、Celery のような分散基盤ではない。

## 基本の書き方（コード）
```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

def write_log(message: str):                 # 普通の関数でOK（def / async def 両方可）
    with open("log.txt", mode="a") as f:
        f.write(message + "\n")

def send_welcome_email(email: str):
    # 実際は SMTP / メールAPI を呼ぶ。失敗してもレスポンスには影響しない点に注意
    print(f"send mail to {email}")

@app.post("/users/")
async def create_user(email: str, background_tasks: BackgroundTasks):  # ★ 引数で受け取る
    # ... ユーザー作成などの本処理 ...
    background_tasks.add_task(write_log, f"created: {email}")   # ★ レスポンス後に実行
    background_tasks.add_task(send_welcome_email, email)        # 複数積める（登録順に実行）
    return {"status": "created"}             # ← これが先に返り、その後 task が走る
```
```python
# Depends と組み合わせると、依存側からも同じ BackgroundTasks にタスクを積める
from fastapi import Depends

def get_query(background_tasks: BackgroundTasks, q: str | None = None):
    if q:
        background_tasks.add_task(write_log, f"query: {q}")
    return q

@app.get("/search/")
async def search(q: str = Depends(get_query)):
    return {"q": q}
```

## 実務での使い方・定番パターン
- **軽くて即時性が不要なものだけ**：通知メール・監査ログ・キャッシュ温め・軽い後片付け。
- **引数で `BackgroundTasks` を受ける**：FastAPI が自動注入する。`add_task(関数, *args, **kwargs)` で引数も渡せる。
- **Depends からも積める**：同一リクエストで生成された `BackgroundTasks` を依存関数が受け取れば、そこからも `add_task` 可能。
- **失敗してもクライアントには伝わらない前提で設計**：タスク内で**例外を握りつぶさず必ずログ**を残す。リトライは自前実装が必要。
- **重い/確実性が要るものは別基盤へ**：分散・リトライ・永続キューが要るなら **Celery / RQ / arq / Dramatiq** などを使う（FastAPI 外）。

## ハマりどころ / アンチパターン
- **再起動でタスクが消える（最重要）**：`BackgroundTasks` は**プロセス内・メモリ上**。デプロイ/クラッシュ/再起動で**実行前のタスクは失われる**。「確実に届く必要がある処理」を載せてはいけない。
- **長時間・重い処理を載せる**：画像処理・大量集計・外部APIの大量呼び出しなどを置くと、ワーカープロセスを占有し他リクエストに悪影響。→ **Celery / RQ など別プロセスのキュー**へ。
  ```python
  # NG: 重い処理を BackgroundTasks に
  background_tasks.add_task(resize_all_images, user_id)   # プロセス内で長時間専有
  # OK: 別基盤のキューに積む（例: Celery）
  resize_all_images.delay(user_id)                        # ワーカープロセスが処理
  ```
- **確実性・リトライを期待する**：`BackgroundTasks` に再試行やデッドレターは無い。失敗時の保証が要る要件には不適。
- **タスク内例外の握り潰し**：例外はレスポンス済みのため**握り潰されやすい**。必ず try/except でログ。監視が無いと「静かに消える処理」になる。
- **DBセッションの寿命**：リクエストスコープの DB セッションは**レスポンスで閉じている**ことがある。タスク内で使うなら**タスク用に開き直す**（`Depends` のセッションをそのまま使い回さない）→ [database.md](./database.md)。
- **重いタスクを `async def` で書いて await ブロッキング**：イベントループを塞ぐ。非同期の扱いは [async.md](./async.md) 参照。

## 関連
[async.md](./async.md) / [database.md](./database.md) / [error_handling.md](./error_handling.md)
