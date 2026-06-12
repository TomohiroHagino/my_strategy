# 設定（Config / Settings）（FastAPI）

## ひとことで言うと
アプリの**外部から差し込む値**（DB接続先・SECRET_KEY・外部APIキー・デバッグ可否など）を、コードに直書きせず**型安全に1か所へ集約**する仕組み。FastAPIでは **`pydantic-settings` の `BaseSettings`** が定番。環境変数や `.env` から読み、`Depends` で注入する。

## 役割・なぜ必要か
- 環境（ローカル / ステージング / 本番）ごとに値を切り替えたい。コード直書きだと毎回書き換え＝事故のもと。
- **シークレットをソースに残さない**（コミット流出を防ぐ）。値は環境変数や Secrets Manager から。
- `BaseSettings` なら**型変換と必須チェック**が効く。`DEBUG=1` が `bool`、ポートが `int` に自動変換され、不足時は起動時に即エラー（フェイルファスト）。

## 基本の書き方（コード）
```python
# config.py
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    # 環境変数 / .env のキー名は大文字小文字を無視して照合される
    app_name: str = "My API"
    debug: bool = False
    database_url: str                       # 既定値なし＝必須（無いと起動時エラー）
    secret_key: str                         # シークレットは必ず外から
    access_token_expire_minutes: int = 30   # 型変換される（"30" -> 30）

    # .env を読む。余計なキーは無視
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache  # プロセス内で1回だけ生成してキャッシュ（毎リクエスト読み直さない）
def get_settings() -> Settings:
    return Settings()
```

```python
# .env（コミット禁止。.gitignore に必ず追加）
DEBUG=1
DATABASE_URL=postgresql+psycopg://user:pass@localhost:5432/app
SECRET_KEY=change-me-in-production
ACCESS_TOKEN_EXPIRE_MINUTES=60
```

```python
# main.py（Depends で注入する）
from fastapi import Depends, FastAPI
from config import Settings, get_settings

app = FastAPI()


@app.get("/info")
def info(settings: Settings = Depends(get_settings)):
    # settings.debug / settings.app_name のように型付きで参照できる
    return {"app": settings.app_name, "debug": settings.debug}
```

## 実務での使い方・定番パターン
- **`@lru_cache` で1度だけ生成**：`Settings()` は `.env` 読込やパースのコストがある。キャッシュして毎リクエストの再読込を避ける。`Depends(get_settings)` は同じインスタンスを返す。
- **必須は既定値を書かない**：`database_url: str` のように既定値なしにすると、未設定時は**起動時に例外**で気付ける（本番で初めて気付くより安全）。
- **本番は `.env` より環境変数**：コンテナ/クラウドでは `.env` ファイルを置かず、環境変数や Secrets Manager（AWS Secrets Manager / GCP Secret Manager 等）から注入。`BaseSettings` は環境変数を自動で拾う。
- **ネスト設定**：`SettingsConfigDict(env_nested_delimiter="__")` で `DB__HOST` のような階層キーを構造化できる。
- **テスト時の差し替え**：`app.dependency_overrides[get_settings] = lambda: Settings(database_url="sqlite://", secret_key="test")` でテスト用設定に差し替え（→ [testing.md](./testing.md)）。
- **型の活用**：URLは `PostgresDsn` / `AnyUrl`、メールは `EmailStr` 等、Pydanticの型でバリデーションを強められる。

## ハマりどころ / アンチパターン
- **`.env` をコミット**：シークレットがGit履歴に残り流出する。`.gitignore` に `.env` を入れ、リポジトリには**値を空にした `.env.example`** だけ置く。万一コミットしたら履歴ごと消し、鍵は**必ず再生成**。
- **キャッシュ忘れ**：`get_settings()` を `@lru_cache` なしにすると毎リクエストで `.env` を読み直し、地味に遅い。逆に**テストで設定を切り替えたいのにキャッシュが効いて変わらない**こともある → テストは `dependency_overrides` で差し替えるか `get_settings.cache_clear()` を呼ぶ。
- **`os.getenv` 直書きの散在**：あちこちで `os.getenv("SECRET_KEY")` すると型変換も必須チェックも効かず、設定の所在が散らばる。`Settings` に集約する。
- **SECRET_KEY をコード直書き / 既定値で本番運用**：`secret_key: str = "dev-secret"` のような既定値のまま本番に出ると重大な脆弱性。本番は必須化し外部注入する（→ [auth.md](./auth.md)）。
- **`extra` 未設定で起動失敗**：`.env` に未定義キーがあると既定で弾かれることがある。意図的に許すなら `extra="ignore"`。
- **`bool` の罠**：`DEBUG=false` という文字列は環境によって truthy に見えるが、`BaseSettings` は `"false"/"0"` を `False` に正しく解釈する。`os.getenv` 直読みだと `"false"` が truthy になり事故る。

## 関連
[dependency_injection.md](./dependency_injection.md) / [auth.md](./auth.md) / [error_handling.md](./error_handling.md)
