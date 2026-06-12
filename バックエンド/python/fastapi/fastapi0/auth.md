# 認証・認可（OAuth2 Password / JWT）（FastAPI）

## ひとことで言うと
- **認証（Authentication）= 「あなたは誰か」** を確かめる（ログイン）。
- **認可（Authorization）= 「その操作をやってよいか」** を判断する（権限）。
- FastAPI 定番は **`OAuth2PasswordBearer`（トークン受け取り）＋ JWT（自己完結トークン）＋ `passlib`（パスワードハッシュ）** の組み合わせ。

## 役割・なぜ必要か
- API はセッションCookieを使わず、**Bearer トークン（`Authorization: Bearer xxx`）** で認証するのが主流。
- **JWT** はサーバ側にセッションを持たず、トークン自体に「誰か」を署名付きで埋め込む＝スケールしやすい。
- 認証だけでは「ログイン済みなら他人のリソースも触れる」ので、**認可（本人のリソースか）**を必ず別に書く。

## 基本の書き方（コード）
### パスワードハッシュ（passlib + bcrypt）
```python
# security.py
from passlib.context import CryptContext

pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(raw: str) -> str:
    return pwd.hash(raw)                       # 登録時：ハッシュ化して保存

def verify_password(raw: str, hashed: str) -> bool:
    return pwd.verify(raw, hashed)             # ログイン時：照合（平文比較は厳禁）
```

### JWT 発行・検証（python-jose）
```python
# jwt_utils.py
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError      # pip install "python-jose[cryptography]"
from config import settings         # SECRET_KEY は env から（config_settings.md）

ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

def create_access_token(sub: str, scopes: list[str]) -> str:
    payload = {
        "sub": sub,                                            # 主体（例: user id）
        "scopes": scopes,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm=ALGORITHM)

def decode_token(token: str) -> dict:
    # exp 切れ・署名不正は JWTError を送出（呼び出し側で 401 に変換）
    return jwt.decode(token, settings.SECRET_KEY, algorithms=[ALGORITHM])
```

### ログイン（OAuth2PasswordRequestForm でトークン発行）
```python
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from deps import DbSession
import crud
from security import verify_password
from jwt_utils import create_access_token

router = APIRouter()

@router.post("/token")
def login(form: OAuth2PasswordRequestForm = Depends(), db: DbSession = None):
    user = crud.get_user_by_email(db, form.username)   # form は username/password 固定
    if not user or not verify_password(form.password, user.hashed_password):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Incorrect email or password")
    token = create_access_token(sub=str(user.id), scopes=user.scopes)
    return {"access_token": token, "token_type": "bearer"}
```

### get_current_user（Depends で保護）★最重要
```python
# deps.py
from typing import Annotated
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError
import crud
from jwt_utils import decode_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")  # /docs に認可ボタンが出る

def get_current_user(token: Annotated[str, Depends(oauth2_scheme)], db: DbSession):
    cred_exc = HTTPException(
        status.HTTP_401_UNAUTHORIZED, "Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = decode_token(token)          # 署名・exp 検証込み
        user_id = payload.get("sub")
        if user_id is None:
            raise cred_exc
    except JWTError:
        raise cred_exc                         # ← 検証漏れを防ぐ：必ず except で 401
    user = crud.get_user(db, int(user_id))
    if user is None:
        raise cred_exc
    return user

CurrentUser = Annotated["User", Depends(get_current_user)]
```

### 保護エンドポイントと認可（本人のリソースか）
```python
@app.get("/users/me")
def read_me(current_user: CurrentUser):
    return current_user

@app.delete("/posts/{post_id}")
def delete_post(post_id: int, current_user: CurrentUser, db: DbSession):
    post = crud.get_post(db, post_id)
    if post is None:
        raise HTTPException(404, "Not found")
    if post.owner_id != current_user.id:        # ★認可：他人の投稿は消させない（IDOR防止）
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Forbidden")
    crud.delete_post(db, post)
    return {"ok": True}
```

### scopes（細かい権限）
```python
from fastapi import Security
from fastapi.security import SecurityScopes

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token", scopes={"items:read": "閲覧", "items:write": "編集"})

def require_scopes(security_scopes: SecurityScopes, current_user: CurrentUser):
    for scope in security_scopes.scopes:
        if scope not in current_user.scopes:
            raise HTTPException(status.HTTP_403_FORBIDDEN, f"Missing scope: {scope}")
    return current_user

@app.post("/items")
def create_item(user=Security(require_scopes, scopes=["items:write"])):
    return {"ok": True}
```

## 実務での使い方・定番パターン
- **`CurrentUser` 型エイリアス**を全保護エンドポイントで共有（DRY）。`/docs` の「Authorize」で動作確認できる。
- **アクセストークン短命＋リフレッシュトークン**で運用（再ログインを減らしつつ漏洩リスクを抑える）。
- **認可は `crud` 層 or Depends に集約**し、各エンドポイントで「本人か / ロールか」を必ず判定。
- ロールは `user.role`（enum）や scopes で表現。重要操作は都度DBの最新権限を見る。

## ハマりどころ / アンチパターン
- **トークン検証漏れ（最重大）**：`decode_token` を `try/except JWTError` で囲まないと、不正トークンが500になったり素通りする。**必ず401に変換**。
- **認証だけして認可を忘れる（IDOR）**：`get_current_user` を通せば「ログイン済み」だけ。`post.owner_id == current_user.id` 等の**本人チェックが別途必須**。
- **`SECRET_KEY` をコードに直書き**：漏洩=全トークン偽造可能。**env / シークレットマネージャ**で管理し、漏れたら必ずローテーション。
- **`exp` を設定しない**：無期限トークンは漏洩時に永久に有効。必ず有効期限を入れる。
- **パスワード平文保存・平文比較**：必ず `passlib` でハッシュ化。ログにパスワードを出さない。
- **HTTPで運用**：Bearer トークンは盗聴可能。本番は必ず HTTPS。

## 関連
[dependency_injection.md](./dependency_injection.md) … `Depends`/`Security` で保護を注入 / [config_settings.md](./config_settings.md) … `SECRET_KEY` を env で安全に管理
