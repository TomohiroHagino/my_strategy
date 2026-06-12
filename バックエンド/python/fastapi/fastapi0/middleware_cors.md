# ミドルウェア / CORS（FastAPI）

## ひとことで言うと
**全リクエスト/レスポンスを横断して通る共通処理の層**。認証ログ・ヘッダ付与・圧縮・CORS（別オリジンからのアクセス許可）などを、各エンドポイントに書かず**アプリ全体に一括適用**する。FastAPIは ASGI の **Starlette ベース**なので、Starlette のミドルウェアがそのまま使える。

## 役割・なぜ必要か
- ロギング・計測・共通ヘッダ・圧縮など「**どのルートでも同じことをしたい**」処理を1か所にまとめる。
- フロント（React等）が**別オリジン**（例: `localhost:3000` → API は `localhost:8000`）の場合、ブラウザの**同一オリジンポリシー**で API 呼び出しがブロックされる。これを解くのが **CORS** であり、`CORSMiddleware` がその設定を担う。

## 基本の書き方（コード）
```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
import time

app = FastAPI()

# ① 既存ミドルウェア: add_middleware で追加（クラス＋設定を渡す）
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://app.example.com"],  # ★ "*" は本番で避ける
    allow_credentials=True,        # Cookie/Authorization を跨ぐ場合 True
    allow_methods=["*"],           # ["GET", "POST", ...] と絞れる
    allow_headers=["*"],           # 許可するリクエストヘッダ
)
app.add_middleware(GZipMiddleware, minimum_size=1000)  # 1KB超のレスポンスを自動gzip圧縮

# ② 自前ミドルウェア: @app.middleware("http") で関数を登録
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)            # ★ ここで後続処理を実行
    response.headers["X-Process-Time"] = f"{time.perf_counter() - start:.4f}"
    return response                                # ★ Response を必ず返す
```

## 実務での使い方・定番パターン
- **CORS は `add_middleware(CORSMiddleware, ...)` で**：`allow_origins` に**具体的なオリジンを列挙**するのが基本。`allow_credentials=True` のとき `allow_origins=["*"]` は**ブラウザ仕様で無効**になるので注意。
- **オリジンは設定/環境変数から**：開発と本番でフロントの URL が違う。`config_settings.md` の Settings から読み込み、ハードコードしない。
- **既存ミドルウェアを優先**：GZip（圧縮）、`TrustedHostMiddleware`（Host偽装対策）、`HTTPSRedirectMiddleware` など Starlette 標準を使う。車輪の再発明をしない。
- **自前ミドルウェアは横断処理だけに**：リクエストID付与・共通ロギング・計測など。**ビジネスロジックは入れない**（それは依存性注入 `Depends` の領分）。
- 認証・認可は基本 `Depends` で行う → [dependency_injection.md](./dependency_injection.md) / [auth.md](./auth.md)

## ハマりどころ / アンチパターン
- **CORS設定ミスでブラウザがブロック（最頻）**：`Access-Control-Allow-Origin` が一致せず、コンソールに CORS エラー。フロントのオリジンを `allow_origins` に**正確に**（スキーム/ポート込みで）入れる。`http://localhost` と `http://localhost:3000` は別物。
  ```python
  # NG: ポートやスキームがズレている／本番URL漏れ
  allow_origins=["localhost:3000"]          # スキーム無しは不一致
  # OK
  allow_origins=["http://localhost:3000", "https://app.example.com"]
  ```
- **`allow_credentials=True` ＋ `allow_origins=["*"]`**：この組み合わせは**ブラウザが拒否**する。Cookie/認証を跨ぐなら**ワイルドカード不可**、オリジンを明示せよ。
- **ミドルウェアの実行順**：`add_middleware` は**後に追加したものが外側（先に実行）**。CORS は外側に置きたいことが多い。順序で挙動が変わるので意識する。
- **自前ミドルウェアで `await call_next(request)` を呼ばない／Response を返さない**：レスポンスが返らずハングする。必ず `response` を返す。
- **重い処理をミドルウェアに載せる**：全リクエストが遅くなる。重い/非同期な処理は背景タスクや別サービスへ → [background_tasks.md](./background_tasks.md)。
- **`@app.middleware("http")` の登録はアプリ起動前に**：起動後の動的追加は効かない。定義時に登録する。

## 関連
[getting_started.md](./getting_started.md) / [dependency_injection.md](./dependency_injection.md) / [auth.md](./auth.md) / [background_tasks.md](./background_tasks.md)
