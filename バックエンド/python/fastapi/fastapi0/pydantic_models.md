# Pydanticモデル（スキーマ・バリデーション）

## ひとことで言うと
`pydantic.BaseModel` を継承して定義する**データの形（スキーマ）**。型注釈を書くだけで、検証・型変換・JSON入出力・自動ドキュメントが付いてくる。FastAPIではリクエストボディとレスポンスの両方をこれで宣言する。

## 役割・なぜ必要か
- **型ヒントが仕様になる**：`name: str` と書けば「文字列必須」、`age: int | None = None` なら「整数 or 省略可」。この宣言から検証・変換・OpenAPIスキーマが自動生成される。
- 入口（リクエスト）で**不正データを弾く**ので、ハンドラ本体は「正しい型のデータが来ている」前提で書ける（防御コードが減る）。
- 出口（レスポンス）で `response_model` に指定すれば、**余計なフィールドを落とす・型を保証する**。パスワード等の漏洩防止にも効く。→ [request_response.md](./request_response.md)
- ORMのモデル（SQLAlchemy等）とは**別物**。DBの行を表すのがORMモデル、API入出力の形を表すのがPydanticモデル。両者を `from_attributes=True` で橋渡しする。

## 基本の書き方（コード）
```python
from datetime import datetime
from pydantic import BaseModel, Field, EmailStr, field_validator

# --- 入力（リクエスト）用 ---
class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=50)          # 制約付き
    email: EmailStr                                          # 形式検証込み
    age: int | None = Field(default=None, ge=0, le=150)      # 既定値＋範囲
    password: str = Field(min_length=8)

    # フィールド単位のカスタム検証（v2）
    @field_validator("name")
    @classmethod
    def name_not_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("name は空白だけにできません")
        return v.strip()

# --- ネストモデル ---
class Team(BaseModel):
    id: int
    name: str

# --- 出力（レスポンス）用：password を含めない ---
class UserOut(BaseModel):
    id: int
    name: str
    email: EmailStr
    team: Team | None = None          # ネスト
    created_at: datetime

    # ORMオブジェクト（属性アクセス）から変換可能にする
    model_config = {"from_attributes": True}
```

```python
# FastAPIでの使い方：入力は引数の型、出力は response_model
from fastapi import FastAPI
app = FastAPI()

@app.post("/users", response_model=UserOut, status_code=201)
def create_user(payload: UserCreate):
    # payload は検証済み。ORMで保存して ORM オブジェクトを返すと
    # from_attributes=True により UserOut へ自動変換される
    user = save_to_db(payload)        # 返り値は SQLAlchemy の User インスタンス想定
    return user
```

## 実務での使い方・定番パターン
- **入力用と出力用を分ける**：`UserCreate`（password有り）/ `UserOut`（password無し）/ `UserUpdate`（全項目Optional）。1つのモデルを使い回さない。
- **`Field` で制約と既定値**：`min_length` / `max_length` / `ge` / `le` / `default`、`Field(..., description="...")` で説明（ドキュメントに出る）。`...`（Ellipsis）は「必須」を意味する。
- **`field_validator`（v2）でカスタム検証**：単一フィールドは `@field_validator`、複数フィールド横断は `@model_validator(mode="after")`。
- **ネストモデル**でJSONの入れ子を表現。リストは `items: list[Item]`。
- **ORM連携**：`model_config = {"from_attributes": True}` を付けると、辞書でなく**属性アクセスできるオブジェクト**（ORM行など）から生成できる。`UserOut.model_validate(orm_user)` で明示変換も可。
- **シリアライズ**：辞書化は `user.model_dump()`、JSON文字列は `user.model_dump_json()`。`exclude_none=True` / `exclude={"password"}` で出し分け。
- **共通設定の共有**：基底クラスに `model_config` を置いて継承。`alias`（JSONのキー名違い）や `populate_by_name=True` もここで。
- 詳細な検証パターンは → [validation.md](./validation.md)

## ハマりどころ / アンチパターン
- **Pydantic v1 と v2 の差**（最頻出）：
  - `class Config: orm_mode = True` → **`model_config = {"from_attributes": True}`**（または `ConfigDict`）。
  - `.dict()` → **`.model_dump()`**、`.json()` → **`.model_dump_json()`**。
  - `@validator` → **`@field_validator`**（`@classmethod` を併用、`pre=True` は `mode="before"` へ）。
  - `parse_obj()` → **`model_validate()`**。
  - 古い記事のコードをそのまま貼ると動かない/警告が出る。バージョンを必ず確認（`pydantic.VERSION`）。
- **入力モデルをそのまま返す**：`password` 等の機微情報が漏れる。レスポンスは必ず出力用モデル＋`response_model` で絞る。
- **既定値にミュータブルを直書き**：`tags: list = []` は危険。`Field(default_factory=list)` を使う。
- **`Optional` と「必須」の混同**：`x: int | None` は「None許容」だが既定値が無ければ**必須**。省略可にするには `= None` を付ける。
- **検証は「型変換」も伴う**：`age: int` に `"30"` を渡すと既定では 30 に変換される。厳密にしたいなら `Field(strict=True)`。
- **ORMの全カラムをレスポンスに出す**：内部用カラムまで露出する。出力モデルで必要な項目だけ宣言する。

## 関連
[request_response.md](./request_response.md) / [validation.md](./validation.md) / [database.md](./database.md)
