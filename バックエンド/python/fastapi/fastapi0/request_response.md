# リクエスト / レスポンス（FastAPI）

## ひとことで言うと
**リクエストの中身（ボディ）を Pydantic モデルで受け取り、レスポンスの形を `response_model` で宣言する**仕組み。入力検証も出力整形（＝機密フィルタ）も型で行う。

## 役割・なぜ必要か
- POST/PUT で送られる JSON ボディを、**Pydantic モデルを関数の引数に取る**だけで受け取れる。型・必須・形式の検証が自動で走り、ダメなら 422。
- 戻り値は `response_model=` で「**出力スキーマ**」を宣言できる。これが単なる型注釈ではなく、**モデルに無いフィールドを出力から落とす**フィルタとして働く ＝ パスワードハッシュ等の機密が**うっかり漏れるのを構造的に防ぐ**。
- `status_code` で成功時のステータス（201 など）を、`Response` 型でヘッダ・Cookie・生レスポンスを細かく制御できる。「入口（Request）と出口（Response）を型で固める」のが FastAPI 流。

## 基本の書き方（コード）
```python
from fastapi import FastAPI, status
from pydantic import BaseModel, EmailStr

app = FastAPI()

# 入力スキーマ：ボディはこの形で来ることを宣言（検証込み）
class UserIn(BaseModel):
    name: str
    email: EmailStr
    password: str          # 受け取るが…

# 出力スキーマ：password を「持たない」＝レスポンスに出ない
class UserOut(BaseModel):
    id: int
    name: str
    email: EmailStr

# 引数に Pydantic モデルを取る → リクエストボディとして解釈される
# response_model=UserOut で出力を UserOut の形にフィルタ
@app.post("/users", response_model=UserOut, status_code=status.HTTP_201_CREATED)
def create_user(user: UserIn):
    saved = {"id": 1, **user.model_dump()}   # password も含む dict
    return saved   # UserOut に無い password は自動で除外されて返る
```

```python
from fastapi import Response

# Response を引数に取るとヘッダ・Cookie・ステータスを直接いじれる
@app.post("/login")
def login(resp: Response):
    resp.set_cookie(key="session", value="xxx", httponly=True)
    resp.status_code = status.HTTP_200_OK
    return {"ok": True}
```

```python
from fastapi import Query

# 1つのエンドポイントで「ボディ・クエリ・パス」が混在する例
@app.put("/items/{item_id}")          # item_id … パス
def update_item(
    item_id: int,                      # パスパラメータ
    item: UserIn,                      # Pydantic モデル → ボディ
    notify: bool = Query(False),       # クエリ ?notify=true
):
    return {"item_id": item_id, "notify": notify, "name": item.name}
```

## 実務での使い方・定番パターン
- **入力用 / 出力用のモデルを分ける**（`UserIn` と `UserOut`）。入力は password を受け、出力は持たない、という非対称をモデルで表現する。詳細は → [pydantic_models.md](./pydantic_models.md)
- `response_model` は **API の契約**。DB から取った行に余計なカラムがあっても、出力モデルに無ければ落ちる＝**機密の漏洩防止が型で保証**される。`response_model_exclude_unset=True` で未設定値を省くテクもある。
- 作成系は `status_code=201`、削除系は `204`（body 無し）など、**意味に合ったステータス**を付ける。
- 「ボディ・クエリ・パスの区別」は **引数の見た目で決まる**：Pydantic モデル＝ボディ、`{}` に対応＝パス、それ以外＝クエリ（明示は `Query`/`Path`/`Body`）。バリデーション強化は → [validation.md](./validation.md)

## ハマりどころ / アンチパターン
- **DB モデルをそのまま `response_model` に使い、機密を漏らす**：password ハッシュ・内部フラグ等を持つ ORM オブジェクトを出力すると全部出かねない。**出力専用モデル**を必ず別に切る。
- **ボディ/クエリ/パスの取り違え**：単純型（`name: str`）はデフォルトで**クエリ**扱い。`str` 単体をボディにしたいなら `Body(...)` を明示。逆に dict を雑にボディ受けすると検証が効かない。
- **`response_model` を信頼しすぎて検証を省く**：`response_model` は出力の整形であって、入力検証は入力モデル側の責務。両方を分けて考える。
- **204 None Content で body を返す**：`status_code=204` なのに dict を返すと矛盾。204 は body 無しが原則。
- 入力に余計なフィールドが来ても、Pydantic v2 は既定で**無視**する。厳格に弾きたいなら `model_config = ConfigDict(extra="forbid")`。→ [pydantic_models.md](./pydantic_models.md)

## 関連
[pydantic_models.md](./pydantic_models.md) / [validation.md](./validation.md)
