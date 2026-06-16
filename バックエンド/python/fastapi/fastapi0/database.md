# データベース（SQLAlchemy + セッション）（FastAPI）

## ひとことで言うと
- **SQLAlchemy = Python の定番 ORM/SQL ツールキット**。テーブルを Python のクラス（モデル）で表し、SQL を書かずに CRUD できる。
- FastAPI 標準の DB 連携は **「SQLAlchemy のセッションを `Depends` で注入する」** のが王道。
- 2.0 系では `Mapped` / `mapped_column` を使った**型注釈ベースのモデル定義**が新標準。

## 役割・なぜ必要か
- リクエストごとに **DBセッション（接続の作業単位）** を1つ開き、処理が終わったら**必ず閉じる**必要がある。
- これを各エンドポイントで手書きすると閉じ忘れ・例外時リークが起きる。**`Depends` + `yield`** で開閉を一元化する。
- ORM があることでSQLインジェクションを避けやすく（パラメータ化される）、モデル＝スキーマで型安全になる。

## 基本の書き方（コード）
### エンジン・セッション・Base（SQLAlchemy 2.0 同期）
```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase

# SQLite は同一スレッド制約があるため check_same_thread=False が必要
engine = create_engine("sqlite:///./app.db", connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):   # 2.0 流の宣言ベース
    pass
```

### モデル定義（2.0 の型注釈スタイル）
```python
# models.py
from sqlalchemy import String, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from database import Base

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    posts: Mapped[list["Post"]] = relationship(back_populates="owner")

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    owner: Mapped["User"] = relationship(back_populates="posts")
```

### get_db 依存（Depends + yield でセッション開閉）★最重要
```python
# deps.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session
from database import SessionLocal

def get_db():
    db = SessionLocal()
    try:
        yield db          # ← ここでエンドポイントに渡す
    finally:
        db.close()        # ← レスポンス後に必ず閉じる（例外時も通る）

DbSession = Annotated[Session, Depends(get_db)]  # 型エイリアスで使い回す
```

### エンドポイントで使う
```python
# main.py
from fastapi import FastAPI, HTTPException
from sqlalchemy import select
from deps import DbSession
import models

app = FastAPI()

@app.get("/users/{user_id}")
def read_user(user_id: int, db: DbSession):
    user = db.get(models.User, user_id)          # 2.0: db.get で主キー取得
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return {"id": user.id, "email": user.email}

@app.post("/users")
def create_user(email: str, db: DbSession):
    user = models.User(email=email)
    db.add(user)
    db.commit()           # 明示的に commit（autocommit=False なので必須）
    db.refresh(user)      # 採番された id 等を読み戻す
    return {"id": user.id}
```

## Alembic（マイグレーション）
モデル変更をDBスキーマに反映する仕組み。**手でテーブルを作らずバージョン管理する**。
```bash
pip install alembic
alembic init alembic                 # alembic/ と alembic.ini を生成
# alembic/env.py で target_metadata = Base.metadata を設定し、Base と models を import
alembic revision --autogenerate -m "create users and posts"  # 差分から雛形生成
alembic upgrade head                 # 最新まで適用
alembic downgrade -1                 # 1つ戻す
```
> `--autogenerate` は**全モデルが import 済み**でないと差分を取りこぼす。`env.py` で確実に読み込む。

## 非同期（async SQLAlchemy / asyncpg）
高並列・I/O待ちが多いなら非同期版。ドライバは PostgreSQL なら `asyncpg`。
```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as db:   # async with で開閉
        yield db

# エンドポイントも async def に。クエリは await する
@app.get("/users/{uid}")
async def read_user(uid: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(models.User, uid)
    return user
```
> 同期エンジンと非同期エンジンは**混在不可**。プロジェクト方針としてどちらかに統一する。

## 実務での使い方・定番パターン
- **`Annotated[Session, Depends(get_db)]`** を型エイリアス化して全エンドポイントで共有（DRY）。
- **CRUD 関数を別モジュール**（`crud.py`）に切り出し、エンドポイントは薄く保つ（リポジトリ的に）。
- 一覧は必ず `.limit()/.offset()` で**ページネーション**（無制限取得を避ける）。
- リレーション読み込みは `selectinload`/`joinedload` で**N+1を潰す**（戦略の使い分けは下記）。

### N+1 とローディング戦略の使い分け
relationship の既定は `lazy="select"`＝**アクセス毎にクエリ＝N+1の元**。`.options(...)` で eager 化する。
```python
from sqlalchemy.orm import selectinload, joinedload
# コレクション（1対多/多対多）→ selectinload（別クエリ IN 取得・行が増殖しない）
stmt = select(User).options(selectinload(User.posts))
# 1対1/多対1 → joinedload（JOINで1クエリ）。※コレクションに joinedload を使うと行が重複
stmt = select(Post).options(joinedload(Post.owner))
# ネスト
select(User).options(selectinload(User.posts).selectinload(Post.comments))
```
- **コレクションに `joinedload` を使うと結果行が増殖**（直積）→ コレクションは `selectinload` が基本。`.unique()` が要る場合もある。
- 強制検出：relationship に **`lazy="raise"`** を設定すると、eager 指定漏れの遅延ロードが例外になり炙り出せる。
- **async 特有**：非同期セッションでは**暗黙の遅延ロードが例外**（`MissingGreenlet`）になる。関連は必ず `selectinload`/`joinedload` で**事前に**ロードしておく。

## ハマりどころ / アンチパターン
- **セッション開閉ミス（最頻）**：`get_db` を `yield` で書かず `return` にすると `finally` が走らず**接続リーク**。必ず `try/finally`（または `with`）。
- **N+1 問題**：ループ内で `post.owner` を都度参照→クエリ大量発行。`select(Post).options(selectinload(Post.owner))` でまとめて取る。
- **commit 忘れ**：`autocommit=False` のため `db.add` だけでは保存されない。**`db.commit()` が必須**。
- **sync/async 混在**：`async def` エンドポイント内で同期セッションをブロッキング呼び出し→イベントループを止めて逆に遅い。エンジンと関数の同期/非同期を揃える。
- **`Depends` 外でセッション生成**：グローバルに1つだけ作って共有→スレッド/リクエスト間で状態が壊れる。必ずリクエストごとに生成。
- **SQLite を本番で**：開発用。本番は PostgreSQL（同期は `psycopg`、非同期は `asyncpg`）。

### SQLAlchemy 特有の現象と対策（N+1以外）
| 現象 | なぜ | 対策 |
|---|---|---|
| **`DetachedInstanceError`** | 既定 `expire_on_commit=True` で **commit後に属性が expire** → セッションclose後にアクセスすると再ロードしようとするが session が無い | `sessionmaker(expire_on_commit=False)`、または **close前に必要属性をロード/DTO(Pydantic)化**。FastAPIなら依存性のセッション生存中にレスポンス用へ詰め替え |
| 結果の取り出し方 | 2.x の `session.execute(select(...))` は Row を返す | エンティティ列なら **`.scalars().all()`**、`joinedload` でコレクションを取った時は **`.unique().scalars()`** |
| commit後に最新値が要る | `add`/`commit` 後、サーバ生成値（IDや default）が古い場合 | `db.refresh(obj)` で再読込 |
| コネクションプール枯渇 | 既定プールに上限。サーバレス/高並行で枯渇 | `create_engine(..., pool_size=, max_overflow=, pool_pre_ping=True)` を調整 |
| 同一性マップの混乱 | 同一セッション内で同じ主キーは**同一インスタンス**を返す（一次キャッシュ） | セッションはリクエストスコープで短命に。長寿命セッションを避ける |

## 関連
[dependency_injection.md](./dependency_injection.md) … `Depends`/`yield` の仕組み（get_db の土台） / [pydantic_models.md](./pydantic_models.md) … モデル↔スキーマ変換（`from_attributes`）
