# FastAPI（Python の高速APIフレームワーク）

## 一言で
**型ヒント駆動の、モダンで高速な API フレームワーク**。Pydantic による自動バリデーション、`async` ネイティブ、そして**コードからSwagger(OpenAPI)ドキュメントが自動生成**されるのが最大の特徴。API構築の新定番。

## 特徴
- **型ヒントが仕様になる**：関数の引数・戻り値の型から、検証・変換・ドキュメントが**自動生成**される。
- **Pydantic モデル**：リクエスト/レスポンスの形をクラスで宣言＝バリデーション込み。
- **async ネイティブ**：`async def` で非同期I/Oを素直に書ける（ASGI / Uvicorn）。
- **依存性注入（`Depends`）**：DBセッション・認証などを宣言的に注入できる（FastAPIの肝）。
- **自動ドキュメント**：`/docs`(Swagger UI)・`/redoc` が勝手に出る。

## どういう使い方をするのか
- **REST API / マイクロサービス**（フロントは別：React等）。
- **ML/データのモデル推論API**（Python資産と相性が良い）。
- 高スループットな非同期API。

## 強み / 弱み
- 強み：開発速度・型安全・自動ドキュメント・高性能。
- 弱み：フルスタックではない（admin/テンプレート等は別途）。非同期を誤用すると逆に遅くなる。

## このフォルダの構成（フラッグシップ＝項目別）
> FastAPIは 0.x 系で大きな版断絶が無いため、版フォルダは作らずここに直接置く（Python 3.9+ 想定）。

### はじめに / コア
- [getting_started.md](./getting_started.md) … 始め方（pip / uvicorn / 自動ドキュメント）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [routing.md](./routing.md) … パスオペレーション（ルーティング・path/query パラメータ）
- [request_response.md](./request_response.md) … リクエストボディ / `response_model` / ステータス
- [pydantic_models.md](./pydantic_models.md) … Pydanticモデル（スキーマ・バリデーション）
- [dependency_injection.md](./dependency_injection.md) … 依存性注入 `Depends`（FastAPIの肝）
- [async.md](./async.md) … `async`/`await`（いつ非同期にすべきか）

### データ・認証
- [database.md](./database.md) … DB（SQLAlchemy＋セッションをDependsで）
- [auth.md](./auth.md) … 認証（OAuth2 password / JWT / セキュリティ）
- [validation.md](./validation.md) … バリデーション（path/query/body の検証）

### 周辺・運用
- [middleware_cors.md](./middleware_cors.md) … ミドルウェア / CORS
- [background_tasks.md](./background_tasks.md) … バックグラウンドタスク（重い処理はCelery）
- [openapi_docs.md](./openapi_docs.md) … 自動ドキュメント（OpenAPI / Swagger / ReDoc）
- [config_settings.md](./config_settings.md) … 設定（pydantic-settings / .env）
- [error_handling.md](./error_handling.md) … 例外処理（HTTPException / ハンドラ）
- [testing.md](./testing.md) … テスト（TestClient / pytest）
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ

> 関連: 同じPythonのフルスタック版は [../django/](../../django/)。環境管理は [../環境管理.md](../../環境管理.md)。
