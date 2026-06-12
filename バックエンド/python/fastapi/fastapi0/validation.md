# バリデーション（path / query / body の検証）（FastAPI）

## ひとことで言うと
- **FastAPI は Pydantic を使って、入力を「型ヒントどおりに自動検証・自動変換」する**。
- 検証用クラス（Pydanticモデル）や `Query()/Path()/Body()` の制約を宣言するだけで、**不正入力は自動で 422** を返す。
- バリデーションロジックを自分で if 文で書かない＝「宣言＝仕様＝ドキュメント」になる。

## 役割・なぜ必要か
- 外部入力（クエリ・パス・ボディ）は**信用できない**。境界で必ず検証するのが鉄則。
- 手書き検証は漏れ・重複・ドキュメント不一致を生む。FastAPI は**型と制約から検証・変換・OpenAPI を一括生成**する。
- 失敗時のレスポンス形式（422＋詳細）が統一されるので、フロント側の扱いも安定する。

## 基本の書き方（コード）
### 型ヒントだけで自動検証・自動変換
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    # /items/abc → item_id が int に変換できず自動で 422
    # /items/3   → item_id=3（int に変換済み）, q は省略可
    return {"item_id": item_id, "q": q}
```

### Query() / Path() で制約を付ける
```python
from typing import Annotated
from fastapi import Query, Path

@app.get("/products")
def list_products(
    # 文字数・正規表現の制約
    keyword: Annotated[str, Query(min_length=2, max_length=50, pattern="^[a-zA-Z0-9 ]+$")],
    # 数値範囲：gt(より大) / ge(以上) / lt / le
    page: Annotated[int, Query(ge=1)] = 1,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,   # 上限を必ず付ける（無制限取得防止）
):
    return {"keyword": keyword, "page": page, "limit": limit}

@app.get("/users/{user_id}")
def get_user(user_id: Annotated[int, Path(gt=0)]):     # 1 以上のみ許可
    return {"user_id": user_id}
```

### Body（Pydanticモデル）でリクエストボディを検証
```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    email: EmailStr                                    # メール形式を自動検証
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=0, le=150)
    tags: list[str] = Field(default_factory=list, max_length=10)

@app.post("/users")
def create_user(payload: UserCreate):   # JSON ボディ → UserCreate に検証・変換
    # ここに来た時点で payload は必ず妥当（型・制約すべて満たす）
    return payload
```

### Body() で単一フィールドや埋め込みを制御
```python
from fastapi import Body

@app.post("/notes")
def create_note(
    title: Annotated[str, Body(min_length=1)],
    content: Annotated[str, Body(max_length=5000)],
    importance: Annotated[int, Body(ge=1, le=5)] = 3,
):
    return {"title": title, "importance": importance}
```

### カスタムバリデータ（field_validator / model_validator）
```python
from pydantic import BaseModel, field_validator, model_validator

class SignUp(BaseModel):
    password: str
    password_confirm: str

    @field_validator("password")             # 単一フィールドの検証
    @classmethod
    def strong_password(cls, v: str) -> str:
        if len(v) < 8 or v.isalpha():
            raise ValueError("8文字以上で英字以外も含めてください")
        return v

    @model_validator(mode="after")           # 複数フィールドにまたがる検証
    def passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError("パスワードが一致しません")
        return self
```
> `field_validator` 内で `raise ValueError(...)` すると、FastAPI が**自動で 422** に変換し、メッセージを `msg` に入れてくれる。

## 検証失敗のレスポンス（自動 422）
不正入力時、FastAPI は **HTTP 422 Unprocessable Entity** と詳細を返す。形は固定なのでフロントで扱いやすい。
```jsonc
// POST /users に {"email": "bad", "age": -1} を送った場合
{
  "detail": [
    {
      "type": "value_error",
      "loc": ["body", "email"],        // どこが（body の email）
      "msg": "value is not a valid email address",
      "input": "bad"
    },
    {
      "type": "greater_than_equal",
      "loc": ["body", "age"],
      "msg": "Input should be greater than or equal to 0",
      "input": -1
    }
  ]
}
```
- `loc` は **「場所の配列」**（`["query", "page"]` / `["path", "user_id"]` / `["body", "field"]`）。
- 複数エラーは**まとめて**返る（最初の1件で止まらない）。

## 実務での使い方・定番パターン
- **リクエスト用モデルとレスポンス用モデルを分ける**（`UserCreate` / `UserRead`）。`response_model` で出力もフィルタ。
- 一覧の `limit` には必ず `le=100` 等の**上限**を付ける（無制限クエリ防止）。
- メール・URL は `EmailStr` / `HttpUrl` 等の**専用型**を使う（自前正規表現より堅い）。
- 422 のメッセージを利用者向けに整えたい場合は `RequestValidationError` のハンドラを上書き（error_handling.md）。

## ハマりどころ / アンチパターン
- **422 の形を知らずにフロントが落ちる**：エラーは `detail` が**配列**。単一文字列だと思って `detail.toLowerCase()` 等をすると壊れる。`loc/msg` をループで扱う。
- **型強制の挙動を誤解**：`item_id: int` なら `"3"` は 3 に変換されるが `"abc"` は 422。クエリの真偽値は `"true"/"1"/"yes"` 等が `True` に変換される点に注意。
- **`Optional` と必須の混同**：`q: str`（必須）と `q: str | None = None`（任意）は別物。デフォルト値の有無で必須/任意が決まる。
- **制約の置き場所ミス**：`Field()` は Pydanticモデル内、`Query()/Path()/Body()` は関数引数。混ぜると効かない。
- **`raise ValueError` 以外で失敗させる**：バリデータ内で `assert` や独自例外を投げると 422 にならず 500 になることがある。検証失敗は `ValueError` で。

## 関連
[pydantic_models.md](./pydantic_models.md) … モデル定義・`Field`・`response_model` の詳細 / [error_handling.md](./error_handling.md) … 422 ハンドラ上書き・`HTTPException`
