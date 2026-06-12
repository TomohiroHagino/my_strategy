# ルーティング（FastAPI）

## ひとことで言うと
**「どの URL・どの HTTP メソッドに、どの関数を割り当てるか」**を決める仕組み。`@app.get/post/put/delete` のデコレータ（＝パスオペレーション）でパスと関数を結びつける。

## 役割・なぜ必要か
- HTTP リクエストの入口を定義する。`GET /items/5` が来たら「どの関数を、どんな引数で呼ぶか」を FastAPI が解決する、その対応表がルーティング。
- FastAPI の特徴は **パス/クエリのパラメータを「関数の引数」として宣言**すること。型ヒントを書けば、文字列で届いた値を **自動で型変換し、失敗したら 422 を返す**（検証もタダで付く）。
- アプリが育つと `main.py` 1ファイルでは破綻するので、**`APIRouter` で機能ごとにルートを分割**し、`tags` で `/docs` 上のグルーピングまで管理できる。

## 基本の書き方（コード）
```python
from fastapi import FastAPI

app = FastAPI()

# パスパラメータ：{item_id} を引数 item_id: int で受ける（自動で int 変換・検証）
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}        # "/items/abc" は int 変換に失敗 → 422

# クエリパラメータ：パスに無い引数は自動でクエリ扱い（?skip=0&limit=10）
@app.get("/items/")
def list_items(skip: int = 0, limit: int = 10):  # デフォルト値があれば任意
    return {"skip": skip, "limit": limit}

# HTTP メソッドごとにデコレータが分かれている
@app.post("/items/")
def create_item(name: str):
    return {"name": name}

@app.put("/items/{item_id}")
def update_item(item_id: int, name: str):
    return {"item_id": item_id, "name": name}

@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    return {"deleted": item_id}
```

```python
# routers/users.py  ── APIRouter でルートを分割する
from fastapi import APIRouter

# prefix で共通パス、tags で /docs 上のグループ名を指定
router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")        # 実体は GET /users/
def list_users():
    return [{"id": 1}]

@router.get("/{user_id}")   # GET /users/{user_id}
def get_user(user_id: int):
    return {"id": user_id}
```

```python
# main.py ── 分割したルータを取り込む
from fastapi import FastAPI
from routers import users

app = FastAPI()
app.include_router(users.router)   # /users 配下が有効になる
```

## 実務での使い方・定番パターン
- **リソース単位で `APIRouter` を切る**（`users.py` / `items.py` / `orders.py`）。`prefix` と `tags` を router 側にまとめておくと、`include_router` 一行で繋がる。
- パスパラメータの型で**入口バリデーションが完結**する（`item_id: int` で非数値を弾く）。さらに範囲（`gt`/`le`）や文字列長を縛りたいときは `Path`/`Query` を使う。→ [validation.md](./validation.md)
- クエリパラメータは「デフォルト値あり＝任意」「デフォルト値なし＝必須」。検索・ページング（`skip`/`limit`）の定番。
- 共通の認証や DB セッションは各ルートに直書きせず `dependencies=` / `Depends` で注入する。→ request/response の組み立ては [request_response.md](./request_response.md)

## ハマりどころ / アンチパターン
- **固定パスより動的パスを先に定義してしまう**：`/users/{user_id}` を `/users/me` より上に書くと、`/users/me` まで `{user_id}="me"` として捕まり、int 変換で 422。**固定パス（`/me`）を動的パス（`/{user_id}`）より先に**書く（上から順にマッチするため）。
- **422 を「サーバのバグ」と勘違い**：パス/クエリの型変換・検証に失敗すると 422（Unprocessable Entity）が返る。これは仕様どおりの入力エラー。レスポンス body に「どの項目が・なぜ」ダメかが出る。
- **末尾スラッシュのズレ**：`@app.get("/items")` と `@app.get("/items/")` は別物。リダイレクト（307）が挟まると POST body が落ちることもあるので、チーム内で有無を統一する。
- **`include_router` の呼び忘れ**：ルータを書いても `app.include_router(...)` しないと `/docs` にも出ず 404。
- 同じパス＋同じメソッドを二重定義すると、**後勝ち**で先のルートが死ぬ（気づきにくい）。

## 関連
[request_response.md](./request_response.md) / [pydantic_models.md](./pydantic_models.md)
