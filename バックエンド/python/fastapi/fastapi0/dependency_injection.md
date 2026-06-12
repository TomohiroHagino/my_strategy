# 依存性注入 `Depends`（FastAPIの肝）

## ひとことで言うと
パスオペレーション（ルート関数）が必要とするもの（DBセッション・認証ユーザ・共通パラメータ等）を、**`Depends()` で宣言的に注入する仕組み**。「この関数を呼んで結果を引数に入れておいて」とFastAPIに頼む書き方。

## 役割・なぜ必要か
- **共通処理の再利用**：DB接続・認証・ページング等を依存関数にまとめ、各エンドポイントで `Depends()` するだけ。コピペが消える。
- **関心の分離**：ルート関数は「本来やりたいこと」に集中し、準備（接続を開く・トークンを検証する）は依存側へ寄せる。
- **後処理の保証（`yield` 依存）**：`yield` を使うと、レスポンス後に必ずクリーンアップ（DBクローズ等）が走る。`try/finally` 相当を宣言的に書ける。
- **テスト容易性**：`app.dependency_overrides` で依存を差し替えられる（本物DB→テスト用、認証→ダミーユーザ）。→ [testing.md](./testing.md)
- 依存は**ネスト**できる（依存が別の依存を `Depends` する）。FastAPIが依存グラフを解決し、同一リクエスト内では既定でキャッシュする。

## 基本の書き方（コード）
```python
from typing import Annotated
from fastapi import Depends, FastAPI, Header, HTTPException
from sqlalchemy.orm import Session

app = FastAPI()

# --- (1) yield依存：DBセッション。後処理（close）が必ず走る ---
def get_db():
    db = SessionLocal()                 # SQLAlchemy のセッション生成
    try:
        yield db                        # ここで値を渡す
    finally:
        db.close()                      # レスポンス後に必ず実行（クリーンアップ）

# --- (2) サブ依存：トークン検証 → 認証ユーザを返す ---
def get_current_user(
    db: Annotated[Session, Depends(get_db)],   # 依存が依存を使う（ネスト）
    x_token: Annotated[str, Header()],
) -> "User":
    user = db.query(User).filter_by(token=x_token).first()
    if user is None:
        raise HTTPException(status_code=401, detail="認証エラー")
    return user

# --- (3) 共通パラメータ依存：ページング ---
def pagination(skip: int = 0, limit: int = 20) -> dict:
    return {"skip": skip, "limit": min(limit, 100)}

# --- 注入して使う（Annotated で型と Depends を併記するのが推奨） ---
@app.get("/me")
def read_me(user: Annotated["User", Depends(get_current_user)]):
    return {"id": user.id, "name": user.name}

@app.get("/items")
def list_items(
    db: Annotated[Session, Depends(get_db)],
    page: Annotated[dict, Depends(pagination)],
):
    return db.query(Item).offset(page["skip"]).limit(page["limit"]).all()
```

## 実務での使い方・定番パターン
- **DBセッションは `yield` 依存**が定番。`get_db` を全エンドポイントで `Depends` し、close を一元化する。→ [database.md](./database.md)
- **認証は依存で**：`get_current_user` を作り、保護したいルートに `Depends`。OAuth2/JWT もこの形に乗る。→ [auth.md](./auth.md)
- **依存の再利用**：`Annotated[Session, Depends(get_db)]` を型エイリアス（`DbDep = Annotated[...]`）にして使い回すと記述が短くなる。
- **値を使わない依存**（副作用だけ）：ルート関数の引数に入れず、デコレータ側に `@app.get("/x", dependencies=[Depends(verify_api_key)])` と書ける（権限チェック・レート制限の門番）。
- **クラス依存**：`Depends(SomeClass)` でインスタンスを注入（設定オブジェクト等）。
- **キャッシュ**：同一リクエスト内で同じ依存は1回だけ実行される。毎回呼びたいなら `Depends(get_x, use_cache=False)`。
- **アプリ全体への適用**：`FastAPI(dependencies=[Depends(...)])` や `APIRouter(dependencies=[...])` でまとめて掛けられる。

## ハマりどころ / アンチパターン
- **`yield` 依存の例外処理**：`yield` の後（finally等）は**レスポンス送信後**に走る。ここで `HTTPException` を投げてもクライアントには既に返答済みで届かない。検証エラーは `yield` の**前**で投げる。
- **`yield` 依存で例外を握り潰す**：`try/except` で握って `finally` だけ close すると、本来のエラーが消える。`finally` でクリーンアップ、例外は再送出が基本。
- **重い依存の使い回しミス**：DB接続プールやクライアントは**毎リクエスト生成しない**。生成コストの高いものはアプリ起動時（lifespan / シングルトン）に作り、依存ではそれを**返すだけ**にする。
- **`Depends()` の付け忘れ**：`user: User = get_current_user` のように直接代入しても呼ばれない。必ず `Depends(get_current_user)` で包む。
- **同期 `yield` 依存の中でブロッキングI/O**：イベントループを塞ぐ可能性。非同期にすべきか、スレッドプール任せ（`def`）かを意識する。→ [async.md](./async.md)
- **巨大な「神依存」**：1つの依存に詰め込みすぎると差し替え・テストが困難。小さく分けてネストで組む。
- **テスト時に override し忘れ**：本物のDB/外部APIを叩いてしまう。`app.dependency_overrides[get_db] = ...` で必ず差し替える。

## 関連
[database.md](./database.md) / [auth.md](./auth.md) / [async.md](./async.md) / [testing.md](./testing.md)
