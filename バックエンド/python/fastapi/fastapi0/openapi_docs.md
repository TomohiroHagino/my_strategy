# 自動ドキュメント（OpenAPI / Swagger / ReDoc）（FastAPI）

## ひとことで言うと
**コードから API 仕様（OpenAPI スキーマ）が自動生成される**仕組み。型ヒントと Pydantic モデルから JSON 仕様が作られ、それを使って **Swagger UI（`/docs`）** と **ReDoc（`/redoc`）** の対話的ドキュメントが**何もせず勝手に出る**。FastAPI 最大の魅力の一つ。

## 役割・なぜ必要か
- 「実装」と「ドキュメント」が**乖離しない**：仕様書を別管理せず、コードが唯一の真実になる。
- フロント/別チームが `/docs` を見れば**そのまま叩いて試せる**（パラメータ入力 → 実行 → レスポンス確認）。
- 生成された OpenAPI JSON（`/openapi.json`）から**クライアントSDKを自動生成**でき、連携が楽になる。

## 基本の書き方（コード）
```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(
    title="My API",
    version="1.0.0",
    description="社内向けユーザーAPI",   # /docs トップに表示される
)

class User(BaseModel):                 # ★ スキーマは Pydantic から自動で作られる
    id: int = Field(..., examples=[1])
    name: str = Field(..., examples=["田中太郎"])
    email: str

@app.get(
    "/users/{user_id}",
    response_model=User,
    tags=["users"],                    # ドキュメント上のグループ分け
    summary="ユーザー1件取得",          # 一覧に出る短い見出し
    description="user_id を指定して1件返す。存在しなければ404。",  # 詳細説明
    responses={                        # ステータス別のレスポンス例を明示
        200: {"description": "成功"},
        404: {"description": "ユーザーが存在しない"},
    },
)
async def get_user(user_id: int):
    """docstring も description として拾われる（description 未指定時）。"""
    return {"id": user_id, "name": "田中太郎", "email": "a@example.com"}
```
```python
# 本番でドキュメントを隠す（None でエンドポイント自体を無効化）
import os
docs_url = "/docs" if os.getenv("ENV") != "production" else None
app = FastAPI(docs_url=docs_url, redoc_url=None, openapi_url=docs_url and "/openapi.json")
```

## 実務での使い方・定番パターン
- **`tags` でグループ化**：`tags=["users"]` のようにまとめると `/docs` が見やすくなる。エンドポイントが増えるほど効く。
- **`summary` / `description` を書く**：一覧の見出しと詳細説明。docstring も `description` に自動採用される。
- **`response_model` で出力を宣言**：レスポンスの形が `/docs` に載り、かつ**余計なフィールドが除去**される（漏洩防止にも有効）→ [request_response.md](./request_response.md)。
- **`examples` で例を提供**：Pydantic の `Field(..., examples=[...])` や `model_config` で、`/docs` の「Try it out」に初期値が入り試しやすい。
- **OpenAPI JSON を活用**：`/openapi.json` をクライアント生成や契約テストに使う。

## ハマりどころ / アンチパターン
- **本番で `/docs` を晒したまま（最重要）**：内部APIの全構造が誰でも見えてしまう。本番では `docs_url=None` で隠すか、**認可をかける**。
  ```python
  # NG: 本番でも誰でも /docs にアクセスできる
  app = FastAPI()                       # docs_url 既定 = "/docs"
  # OK: 環境で出し分け、または Depends で保護
  app = FastAPI(docs_url=None if PROD else "/docs")
  ```
- **`docs_url=None` だけ消して `openapi_url` を残す**：`/openapi.json` から仕様が丸見えのまま。隠すなら `openapi_url=None` も併せて無効化（ただし無効化すると `/docs` も動かない点に注意）。
- **スキーマのカスタマイズ方法を知らない**：認証スキームや共通レスポンスを足したいときは `app.openapi` を上書き（`get_openapi` で生成 → 加工 → キャッシュ）する。デコレータ引数だけでは限界がある。
- **レスポンス例とモデルが食い違う**：`response_model` と手書き `responses` の例がズレるとドキュメントが嘘になる。基本は `response_model` に寄せる。
- **`tags`/`summary` 無しで肥大化**：エンドポイントが増えると `/docs` が一枚岩で読めなくなる。早めにタグ分割。

## 関連
[pydantic_models.md](./pydantic_models.md) / [getting_started.md](./getting_started.md) / [request_response.md](./request_response.md)
