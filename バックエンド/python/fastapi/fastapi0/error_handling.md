# 例外処理（Error Handling）（FastAPI）

## ひとことで言うと
APIが**エラーをどう表現してクライアントに返すか**の仕組み。FastAPIの基本は **`HTTPException(status_code, detail)`** を投げること。加えて**カスタム例外ハンドラ**（`@app.exception_handler`）で独自例外をJSONに変換し、Pydanticの検証失敗は **`RequestValidationError`（422）** として自動的に返る。

## 役割・なぜ必要か
- エラーは**握りつぶさず、適切なHTTPステータス＋一貫した形のJSON**で返す必要がある。クライアントは status code で分岐するため。
- 例外を放置すると `500 Internal Server Error` になり、**スタックトレースや内部詳細が漏れる**リスク。本番では内部情報を隠し、ログにだけ詳細を残す。
- ドメイン例外（例：在庫不足）を HTTP の都合（409 など）へ**一箇所で変換**でき、各エンドポイントが綺麗になる。

## 基本の書き方（コード）
```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()
items = {"foo": "bar"}


@app.get("/items/{item_id}")
def read_item(item_id: str):
    if item_id not in items:
        # status_code と detail を指定して投げる。FastAPIがJSONに変換
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found",          # クライアントに見せてよい範囲のみ
            headers={"X-Error": "item"},      # 任意でヘッダも付与可能
        )
    return {"item": items[item_id]}
# レスポンス例: 404 {"detail": "Item not found"}
```

```python
# カスタム例外 + ハンドラ（ドメイン例外をHTTPへ一括変換）
from fastapi import Request
from fastapi.responses import JSONResponse


class OutOfStockError(Exception):
    def __init__(self, sku: str):
        self.sku = sku


@app.exception_handler(OutOfStockError)
async def out_of_stock_handler(request: Request, exc: OutOfStockError):
    return JSONResponse(
        status_code=status.HTTP_409_CONFLICT,
        content={"detail": f"在庫切れ: {exc.sku}"},
    )


@app.post("/order/{sku}")
def order(sku: str):
    raise OutOfStockError(sku)   # ハンドラが拾って 409 に変換
```

```python
# バリデーション失敗(422)の形を上書きしたいとき
from fastapi.exceptions import RequestValidationError


@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    # exc.errors() に「どのフィールドがなぜ不正か」が入る
    return JSONResponse(status_code=422, content={"errors": exc.errors()})
```

## 実務での使い方・定番パターン
- **`HTTPException` を素直に使う**：404/401/403/409 など想定内のエラーは `raise HTTPException(...)`。`status.HTTP_*` 定数を使うとマジックナンバーを避けられる。
- **ドメイン例外 → ハンドラ変換**：サービス層は `OutOfStockError` のような**HTTPを知らない例外**を投げ、`@app.exception_handler` で HTTP ステータスに翻訳。層の責務が綺麗に分かれる。
- **全例外の最後の砦**：`@app.exception_handler(Exception)` で予期せぬ例外を捕捉し、**ログには full traceback、レスポンスには汎用メッセージ**（`{"detail": "Internal Server Error"}`）を返す。
- **検証エラー(422)の整形**：フロントが扱いやすいよう `exc.errors()` を整形して返す。デフォルトでも十分情報量がある（→ [validation.md](./validation.md)）。
- **ログ設計**：`logging` で `exc_info=True` を付けてサーバ側に詳細を残す。`request.url` / 相関ID も一緒に記録すると追跡しやすい。

## ハマりどころ / アンチパターン
- **例外の握りつぶし**：`try: ... except Exception: pass` は最悪。エラーが消えてデバッグ不能になり、壊れたまま `200` を返すこともある。**握るなら必ずログ＋適切なステータスで再送**。
- **本番で内部詳細 / トレースが漏れる**：`detail=str(exc)` のように例外文字列をそのまま返すと、SQL文・ファイルパス・スタックトレースが外部に漏れる。**ユーザ向けメッセージと内部ログを分離**する。`debug=True` を本番で有効にしない（→ [config_settings.md](./config_settings.md)）。
- **`HTTPException` を `except` で握ってしまう**：広い `except Exception` がFastAPIの `HTTPException` まで飲み込み、意図した404が500になる。捕捉対象を絞るか、`HTTPException` は再raiseする。
- **ステータスコードの取り違え**：「入力が不正」を 500 で返す等。検証は422、認証は401、認可は403、競合は409 と適切に。
- **`raise` を忘れて `return HTTPException(...)`**：`HTTPException` は**投げる**もの。`return` すると例外オブジェクトがそのままシリアライズされ意図しない結果に。
- **async ハンドラ内でのブロッキング**：ハンドラ内で同期I/Oやブロッキング処理をすると全体が詰まる（→ [pitfalls.md](./pitfalls.md) / [async.md](./async.md)）。

## 関連
[validation.md](./validation.md) / [config_settings.md](./config_settings.md) / [request_response.md](./request_response.md)
