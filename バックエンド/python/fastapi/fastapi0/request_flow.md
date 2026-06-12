# リクエストの流れ・各層は何を返すか（FastAPI）

## ひとことで言うと
1リクエストが **path operation(@app.post) → Depends → crud/service → model(SQLAlchemy)** と降り、**ORMモデルが逆向きに上がってきて、`response_model` の Pydantic スキーマで JSON に整形して返す**。schema=入出力DTO。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
クライアント
   │ リクエスト（request body = JSON）
   ▼
[path operation(@app.post)]  リクエストBody（Pydantic schema）で受け取る＝自動検証
   │
   ▼
[Depends]   DBセッション・認証など依存を注入（宣言的）
   │
   ▼
[crud / service]  業務ロジック／ORM を呼ぶ
   │
   ▼
[model(SQLAlchemy)]  DBアクセス
   │
   ▼
  DB ──→ ORMモデル を返す ─┐
   ▲                        │
[crud/service] が ORMモデル を path operation に返す
   ▲
[path operation] が return → response_model(Pydantic) で JSON に整形して返す
   │ レスポンス（response body = JSON）
   ▼
クライアント
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **path operation** | リクエストBody（Pydantic schema）/ path・query | `response_model` に沿った **JSON** |
| **Depends** | （宣言された依存） | DBセッション・認証ユーザ等を注入 |
| **crud / service** | schema / 値 / セッション | **ORMモデル**（SQLAlchemy） |
| **model(SQLAlchemy)** | クエリ条件 / 保存データ | **ORMモデルインスタンス**（DBから） |
| **schema(Pydantic)** | — | 入出力の形（DTO）。入力=検証、出力=整形 |

- 入力schemaが検証を、出力schema(`response_model`)が整形を担う。境界はPydanticで固める。
- crud/serviceはORMモデルを返し、path operationのreturnで`response_model`がJSON化する。

## コードで通して見る
```python
# schema：入出力DTO（入力PostCreate / 出力PostOut）
class PostCreate(BaseModel):
    title: str
    body: str

class PostOut(BaseModel):
    id: int
    title: str
    model_config = {"from_attributes": True}  # ORMモデルから読めるように

# crud：ORMで作成し、ORMモデルを返す
def create_post(db: Session, data: PostCreate) -> Post:
    post = Post(title=data.title, body=data.body)
    db.add(post); db.commit(); db.refresh(post)
    return post  # ORMモデルを返す

# path operation：schemaで受け取り → Dependsでセッション注入 → crud呼び出し → response_modelで整形
@app.post("/posts", response_model=PostOut)
def post_create(data: PostCreate, db: Session = Depends(get_db)):
    return create_post(db, data)  # ORMモデルを返す→PostOutでJSON化される
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：crud/service→path operation は ORMモデル、path operation→クライアントは `response_model`(Pydantic) で JSON。
- **入力DTOと出力DTOを分ける**：`PostCreate`(入力) と `PostOut`(出力) を別クラスに。内部列の露出を防ぐ。→ [pydantic_models.md](./pydantic_models.md)
- **DBセッション・認証は Depends で注入**：テスト時に差し替えやすい。→ [dependency_injection.md](./dependency_injection.md)
- **`response_model` を必ず付ける**：出力の形が固定され、OpenAPIにも反映される。→ [request_response.md](./request_response.md)

## ハマりどころ / アンチパターン
- **ORMモデルをそのまま返す（response_model無し）**：内部列が漏れる/シリアライズ事故。出力schemaを付ける。
- **`from_attributes`(旧orm_mode) 忘れ**：ORMモデル→Pydantic変換が失敗する。出力schemaに設定。
- **同期DBドライバを `async def` で呼ぶ**：イベントループを塞いで逆に遅い。async混在に注意。→ [async.md](./async.md)
- **業務ロジックを path operation に直書き**：肥大化する。crud/serviceに切り出す。→ [pitfalls.md](./pitfalls.md)

## 関連
[routing.md](./routing.md) / [request_response.md](./request_response.md) / [pydantic_models.md](./pydantic_models.md) / [dependency_injection.md](./dependency_injection.md) / [database.md](./database.md) / [validation.md](./validation.md)
