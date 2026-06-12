# 実務でハマる罠まとめ（Pitfalls）（FastAPI）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- FastAPIは型ヒントと非同期で楽に書ける反面、「便利な書き方」と「事故る書き方」が紙一重。症状から該当箇所へ素早く飛ぶための索引。

## async / 非同期
- **`async def` 内でブロッキング呼び出し → 全体が詰まる**：`time.sleep`・同期DBドライバ・`requests` などをイベントループ上で呼ぶと、その間**他の全リクエストが止まる**。非同期版（`asyncio.sleep` / 非同期DBドライバ / `httpx`）を使うか、重い同期処理は `def`（通常関数）にしてFastAPIにスレッドプール実行させる。→ [async.md](./async.md)
- **何でも `async def` にする誤用**：中身が同期ブロッキングなら逆効果。I/O待ちが非同期化されて初めて意味がある。→ [async.md](./async.md)

## レスポンス / スキーマ
- **`response_model` で機密漏れ防止**：DBモデルをそのまま返すと `password_hash` 等まで漏れる。**出力専用のPydanticモデル**を `response_model=` に指定し、返す項目を絞る。→ [request_response.md](./request_response.md)
- **入力モデルをそのまま出力に流用**：パスワード等を含む入力スキーマを出力に使うと漏洩。入力用/出力用を分ける。→ [request_response.md](./request_response.md) / [pydantic_models.md](./pydantic_models.md)

## DB / セッション
- **`Depends` + `yield` のセッション管理**：DBセッションは `yield` 依存で**確実にclose**する（`try/finally` or `with`）。close忘れはコネクション枯渇の原因。→ [database.md](./database.md)
- **N+1問題**：一覧で関連を1件ずつ引く。SQLAlchemyは `selectinload` / `joinedload` でまとめ読み。→ [database.md](./database.md)
- **同期ドライバを async ルートで使う**：`psycopg2` をそのまま `async def` で呼ぶと詰まる。非同期なら `asyncpg` + async session、または同期ルート(`def`)にする。→ [database.md](./database.md) / [async.md](./async.md)

## バックグラウンド処理
- **`BackgroundTasks` は再起動で消える → Celery**：FastAPI標準の `BackgroundTasks` はプロセス内で動くだけで、**デプロイ/クラッシュ/再起動で未処理タスクは消える**。永続化・リトライ・分散が要るなら Celery / RQ / Arq などのジョブキューへ。→ [background_tasks.md](./background_tasks.md)

## Pydantic / バリデーション
- **Pydantic v1 / v2 の差**：`orm_mode`→`from_attributes`、`.dict()`→`.model_dump()`、`@validator`→`@field_validator`、`Config`クラス→`model_config`。記事のコードがどちらの版か確認。FastAPIの新しめ版はv2前提。→ [pydantic_models.md](./pydantic_models.md)
- **検証エラー(422)の形**：Pydanticの検証失敗は自動で **422** + `detail` にフィールド別エラー。フロントはこの形を前提に実装。整形したいなら `RequestValidationError` ハンドラで上書き。→ [validation.md](./validation.md) / [error_handling.md](./error_handling.md)
- **例外の握りつぶし / 本番でトレース漏れ**：`except: pass` で握る、`detail=str(exc)` で内部情報を返すのは事故。ログと外部レスポンスを分離。→ [error_handling.md](./error_handling.md)

## セキュリティ / 運用
- **本番で `/docs`・`/redoc` を公開**：内部APIの仕様が誰でも見える。本番では `docs_url=None, redoc_url=None` で無効化、または認証で保護。→ [openapi_docs.md](./openapi_docs.md)
- **SECRET_KEY のハードコード / 既定値運用**：JWT署名鍵などを直書き・弱い既定値のまま本番に出ると重大脆弱性。環境変数 / Secrets Manager から注入し、漏れたら必ず再生成。→ [config_settings.md](./config_settings.md) / [auth.md](./auth.md)
- **`.env` をコミット**：シークレットがGit履歴に残り流出。`.gitignore` に追加し `.env.example` だけ置く。→ [config_settings.md](./config_settings.md)
- **CORSを `allow_origins=["*"]` で本番運用**：認証付きAPIでワイルドカード許可は危険。許可オリジンを明示する。→ [middleware_cors.md](./middleware_cors.md)

## テスト
- **テストDB未用意 / 依存オーバーライド忘れ**：`dependency_overrides` を忘れると本物のDBを汚す。`clear()` 忘れで次のテストへリーク。fixtureで前後処理を閉じる。→ [testing.md](./testing.md)

## 関連
[async.md](./async.md) / [database.md](./database.md) / [request_response.md](./request_response.md) / [background_tasks.md](./background_tasks.md) / [config_settings.md](./config_settings.md) / [error_handling.md](./error_handling.md) / [testing.md](./testing.md)
