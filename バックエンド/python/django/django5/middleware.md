# ミドルウェア（middleware）（Django 5）

## ひとことで言うと
**全リクエスト/レスポンスの前後に共通で挟まる処理**の層。View に届く前と、View から返った後の両方に割り込んで、横断的な処理（認証チェック・セキュリティヘッダ付与・ログ等）を一括で行う。

## 役割・なぜ必要か
- ログイン必須チェック、CSRF検証、セキュリティヘッダ、Gzip圧縮、セッション復元など「**どのViewでも共通でやりたいこと**」を、各Viewに書かず1か所に集約する。
- リクエストの流れ：`ブラウザ → ミドルウェア群（行き）→ View → ミドルウェア群（帰り）→ レスポンス`。**玉ねぎ構造**で、行きは上から、帰りは下から処理される。
- `settings.py` の `MIDDLEWARE` リストの**並び順が処理順**になる。順序を間違えると正しく動かない。

## 基本の書き方（コード）
```python
# settings.py … 上が「外側」、リクエストはこの順で通る
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",          # 1 セキュリティヘッダ/HTTPS
    "django.contrib.sessions.middleware.SessionMiddleware",   # 2 セッション復元
    "django.middleware.common.CommonMiddleware",              # 3 URL正規化など
    "django.middleware.csrf.CsrfViewMiddleware",              # 4 CSRF検証
    "django.contrib.auth.middleware.AuthenticationMiddleware",# 5 request.user を用意
    "django.contrib.messages.middleware.MessageMiddleware",   # 6 フラッシュメッセージ
    "django.middleware.clickjacking.XFrameOptionsMiddleware", # 7 クリックジャッキング対策
]
```

```python
# myapp/middleware.py … 自作ミドルウェア（関数を返す呼び出し可能オブジェクト）
import time
import logging

logger = logging.getLogger(__name__)


class TimingMiddleware:
    def __init__(self, get_response):
        # サーバ起動時に1回だけ呼ばれる（初期化）
        self.get_response = get_response

    def __call__(self, request):
        # --- 行き：View 前の処理 ---
        start = time.monotonic()

        response = self.get_response(request)  # 次の層 → 最終的に View を呼ぶ

        # --- 帰り：View 後の処理 ---
        elapsed = (time.monotonic() - start) * 1000
        response["X-Response-Time-ms"] = f"{elapsed:.1f}"
        logger.info("%s %s -> %s (%.1fms)",
                    request.method, request.path, response.status_code, elapsed)
        return response
```

```python
# settings.py に自作ミドルウェアを追加（位置 = 効かせたい順序で決める）
MIDDLEWARE += ["myapp.middleware.TimingMiddleware"]
```

## 実務での使い方・定番パターン
- **`__call__` の構造を覚える**：`get_response(request)` の**前**がリクエスト処理、**後**がレスポンス処理。`return response` を忘れない。
- **標準ミドルウェアの役割**：
  - `SecurityMiddleware`：HTTPSリダイレクト・HSTS・各種セキュリティヘッダ。最上段に置く。
  - `CsrfViewMiddleware`：POST等のCSRFトークン検証。`SessionMiddleware` の後に必要。
  - `CommonMiddleware`：末尾スラッシュ補完（`APPEND_SLASH`）、`DISALLOWED_USER_AGENTS` 等。
  - `AuthenticationMiddleware`：`request.user` を生やす。これより後でないと `request.user` が使えない。
- **Django 5.1 の `LoginRequiredMiddleware`**：**サイト全体をデフォルトでログイン必須**にできる。個別Viewに `@login_required` を貼って回らずに済む。
  ```python
  MIDDLEWARE += ["django.contrib.auth.middleware.LoginRequiredMiddleware"]
  # 公開したいViewだけ明示的に除外する
  from django.contrib.auth.decorators import login_not_required

  @login_not_required
  def public_landing(request):
      ...
  ```
  `AuthenticationMiddleware` より後に置くこと。未ログインなら `LOGIN_URL` へリダイレクトされる。→ [auth.md](./auth.md)
- **フック用メソッド**も使える：`process_view` / `process_exception` / `process_template_response`。例外を共通でハンドリングしたいときは `process_exception(request, exception)`。
- **非同期対応**：async View 主体なら async ミドルウェアにできる。`async_capable` / `sync_capable` フラグや `django.utils.decorators.sync_and_async_middleware` で両対応に。

## ハマりどころ / アンチパターン
- **順序ミスが最頻の罠**。例：`CsrfViewMiddleware` を `SessionMiddleware` より前に置くと検証が壊れる、`AuthenticationMiddleware` より前で `request.user` を触ると AttributeError。**「行きは上から、帰りは下から」**を意識して並べる。
- **`SecurityMiddleware` は最上段**に。下の方に置くとHTTPSリダイレクト前に他処理が走り、ヘッダが意図通り付かない。
- **重い処理を全リクエストに乗せない**。ミドルウェアは**毎リクエスト必ず実行**される。DBアクセス・外部API・重い計算を入れると全ページが遅くなる。本当に全体共通か、Viewやデコレータで足りないかを先に検討する。→ [orm_queries.md](./orm_queries.md)
- **`get_response` の戻りを返し忘れる**と、レスポンスが None になり 500。`return response` を必ず書く。
- **`__init__` は起動時1回だけ**。ここにリクエスト依存の状態を持たせると全リクエストで共有されてしまう（スレッド安全性の事故）。リクエスト固有の値は `__call__` 内のローカル変数か `request` 属性に持つ。
- **`LoginRequiredMiddleware` の除外漏れ**。全体必須にしたのにログインページ自体や公開APIを `login_not_required` で除外し忘れると、ログインできない/外部から叩けない状態になる。
- **`MIDDLEWARE` から外すと機能ごと消える**。セッション・認証・CSRF は他機能の前提。安易に削らない。→ [security.md](./security.md)

## 関連: [auth.md](./auth.md) / [security.md](./security.md)
