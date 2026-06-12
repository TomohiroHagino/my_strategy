# はじめに（FastAPI）

## ひとことで言うと
FastAPI を **インストールして最小アプリを起動し、自動ドキュメント（`/docs`・`/redoc`）まで動かす**ための最初の一歩。型ヒントを書くだけで検証・変換・ドキュメントが付いてくる。

## 役割・なぜ必要か
- まず「**Python のコードを HTTP で叩ける**」状態を最短で作るのが目的。`app = FastAPI()` がアプリ本体、`@app.get("/")` のデコレータでパスに関数を紐づける。
- FastAPI 自身は ASGI アプリの「定義」だけを持ち、実際にリクエストを受けるのは **ASGI サーバ（Uvicorn）**。だから `uvicorn main:app` で起動する、という二段構えになっている。
- 最大の売りは **OpenAPI スキーマの自動生成**。コードを書けば `/docs`（Swagger UI）と `/redoc` が勝手に出るので、API 仕様書を別途書かなくてよい。
- 対象は **Python 3.9+**（型ヒント・`Annotated` などモダンな機能を使うため）。

## 基本の書き方（コード）
```bash
# 仮想環境を作って有効化（推奨）
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# [standard] で uvicorn・httpx 等の定番一式が入る
pip install "fastapi[standard]"
```

```python
# main.py
from fastapi import FastAPI

app = FastAPI()  # ← アプリ本体（ASGI アプリ）

@app.get("/")            # ルート（GET /）
def read_root():
    return {"hello": "world"}   # dict を返すと自動で JSON 化される

@app.get("/items/{item_id}")    # パスパラメータは型ヒントで自動変換・検証
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

```bash
# 起動（main.py 内の app を指す）。--reload は開発時のみ
uvicorn main:app --reload
# → http://127.0.0.1:8000        本体
# → http://127.0.0.1:8000/docs   Swagger UI（試せる）
# → http://127.0.0.1:8000/redoc  ReDoc（読みやすい仕様書）

# fastapi[standard] が入っていれば CLI でも起動できる
fastapi dev main.py        # 開発用（自動リロード付き）
fastapi run main.py        # 本番想定（リロードなし）
```

## 実務での使い方・定番パターン
- **`async def` か `def` か**は「中で何を待つか」で決める。`await` する非同期ライブラリ（httpx・非同期DBドライバ）を使うなら `async def`、同期ライブラリ（requests・通常の SQLAlchemy）なら `def`。判断に迷う詳細は → [async.md](./async.md)
- 開発は `--reload`（または `fastapi dev`）で保存即反映。**本番では `--reload` を外し**、`uvicorn`/`gunicorn` のワーカ数を指定して動かす。
- `/docs` は実装と常に同期した「**生きた仕様書 兼 動作確認ツール**」。フロント担当に URL を渡すだけで連携が進む。→ [openapi_docs.md](./openapi_docs.md)
- ルートが増えてきたら `main.py` に全部書かず、`APIRouter` で機能ごとにファイル分割する。→ [routing.md](./routing.md)

## ハマりどころ / アンチパターン
- **`[standard]` を付け忘れる**：`pip install fastapi` だけだと `uvicorn` も `fastapi` CLI も入らず「コマンドが無い」となる。基本は `"fastapi[standard]"` を入れる（クォート必須＝`zsh` で `[]` が展開されるのを防ぐ）。
- **`uvicorn main:app` の `main:app` を間違える**：`main` は **ファイル名（拡張子なし）**、`app` は **その中の変数名**。`app.py` に `app = FastAPI()` なら `uvicorn app:app`。
- **同期処理を `async def` の中で直接呼ぶ**：`async def` 内で `time.sleep()` や同期DBアクセスをするとイベントループを止め、全体が遅くなる。重い同期処理は `def` で書く（FastAPI が別スレッドに逃がす）。→ [async.md](./async.md)
- **本番で `--reload` を付けっぱなし**：ファイル監視が走り続けて無駄。デプロイ時は外す。
- グローバル Python に直接 `pip install` してバージョンが衝突。**仮想環境（venv）** を切るのが基本。→ [../環境管理.md](../../環境管理.md)

## フォルダ構成（始動直後）
```
myapp/                                    # ※FastAPIに公式雛形は無い。実務でよくある構成
├── app/
│   ├── __init__.py
│   ├── main.py                           # FastAPI() 生成・ルータ登録・起動
│   ├── core/
│   │   ├── config.py                     # 設定（pydantic-settings）
│   │   └── security.py                   # 認証・JWT
│   ├── api/
│   │   ├── __init__.py
│   │   └── v1/
│   │       ├── api.py                    # ルータ集約
│   │       └── endpoints/
│   │           └── users.py              # エンドポイント（path operation）
│   ├── models/                           # DBモデル（SQLAlchemy）
│   │   └── user.py
│   ├── schemas/                          # 入出力スキーマ（Pydantic）
│   │   └── user.py
│   ├── crud/                             # DB操作（取得/作成 …）
│   │   └── user.py
│   ├── db/
│   │   └── base.py  session.py           # DB接続・セッション
│   └── deps.py                           # 依存（Depends で注入）
├── tests/
│   └── test_users.py
├── alembic/                              # マイグレーション（任意）
│   ├── versions/
│   └── env.py
├── alembic.ini
├── requirements.txt （or pyproject.toml） # 依存
└── .env                                  # 環境変数
```
- **公式の決まった構成は無い。** 上は「`api`(受付)/`schemas`(Pydantic)/`models`(DB)/`crud`(DB操作)/`core`(設定)」に分ける定番形。
- 最小なら `main.py` 1枚でも動く。規模に応じてこの形へ育てる。
- DBマイグレーションは **Alembic**（SQLAlchemy系）が定番。

## 関連
[routing.md](./routing.md) / [openapi_docs.md](./openapi_docs.md)
