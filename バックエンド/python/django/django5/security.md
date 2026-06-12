# セキュリティ（Security）（Django 5）

## ひとことで言うと
Webアプリの定番攻撃（CSRF / XSS / SQLインジェクション / クリックジャッキング等）に対する、**Djangoが標準で用意する防御の使い方**。多くは既定で有効だが、「無効化してしまう書き方」（`|safe`・`raw()`・`DEBUG=True` のまま本番）を避けるのがポイント。

## 役割・なぜ必要か
- 外部から来る入力（GET/POST・ヘッダ・Cookie・ファイル名）は**信用できない**前提で扱う必要がある。
- Djangoは安全側のデフォルト（テンプレート自動エスケープ・CSRFトークン・ORMのパラメータ化）を持つが、**素朴な書き方で穴が開く**。どこが守られ、どこで自分が破りうるかを知るのが目的。

## 基本の書き方（コード）
```django
{# 1) CSRF: POSTフォームには必ずトークンを入れる（CsrfViewMiddlewareが検証） #}
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">送信</button>
</form>

{# 2) XSS: テンプレートは既定で自動エスケープ → 基本そのままで安全 #}
{{ user.bio }}            {# < > & " が実体参照に変換され安全 #}
{{ user.bio|safe }}      {# NG: エスケープ無効化。ユーザ入力に付けると即XSS #}
```

```python
# 3) SQLインジェクション: ORMはパラメータ化されるので原則安全
Article.objects.filter(title=request.GET["q"])        # OK（自動でバインド）

# raw() / extra() を使うときも文字列連結しない
from django.db import connection
Article.objects.raw("SELECT * FROM app_article WHERE title = %s",
                    [request.GET["q"]])                # OK（%s でバインド）
# Article.objects.raw(f"... WHERE title = '{q}'")      # NG: 絶対に書かない

# 4) 本番設定（settings.py）
DEBUG = False                                  # 本番は必ず False
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]   # コードに直書きしない
ALLOWED_HOSTS = ["example.com"]                # 本番は具体的に列挙
```

## 実務での使い方・定番パターン
- **CSRF**：`MIDDLEWARE` の `CsrfViewMiddleware`（既定有効）が検証。HTMLフォームは `{% csrf_token %}` を入れれば自動。JS（fetch/axios）から叩くなら Cookie の `csrftoken` をヘッダ `X-CSRFToken` に載せる。DRFはトークン/JWT認証＋`SessionAuthentication`時のみCSRF対象。→ [templates.md](./templates.md) / [drf.md](./drf.md)
- **XSS**：DTLは既定で自動エスケープ。`|safe`・`mark_safe()`・`autoescape off` は整形済み安全文字列にだけ使う。ユーザ入力HTMLを通すなら `bleach` 等でサニタイズ。→ [templates.md](./templates.md)
- **`SECRET_KEY` / `DEBUG` / `ALLOWED_HOSTS`**：3点セットで管理。秘密は環境変数 or `django-environ` で `.env` から読む。`DEBUG=True` のままだと例外画面に設定やトレースが丸見え。→ [settings.md](./settings.md)
- **クリックジャッキング**：`XFrameOptionsMiddleware`（既定有効）が `X-Frame-Options: DENY` を付与。iframe埋め込みを許すページだけ `@xframe_options_exempt`。
- **SecurityMiddleware**：`SECURE_SSL_REDIRECT`（HTTPS強制）、`SECURE_HSTS_SECONDS`、`SESSION_COOKIE_SECURE` / `CSRF_COOKIE_SECURE`、`SECURE_CONTENT_TYPE_NOSNIFF` をまとめて設定。
- **認可**：`request.user.articles.get(pk=...)` のように**常にログインユーザのスコープ**で引き、他人のリソース操作を構造的に防ぐ。ビューには `@login_required` / `LoginRequiredMixin`（5.1の `LoginRequiredMiddleware` で全体一括も可）。→ [auth.md](./auth.md)
- **デプロイ前チェック**：`python manage.py check --deploy` で本番向け設定の不備（`DEBUG`・Cookie secure・HSTS等）を一括検出。CIに入れる。

## ハマりどころ / アンチパターン
- **本番 `DEBUG = True`**：最大級の事故。例外画面に `SECRET_KEY`・環境変数・SQL が露出する。本番は必ず `False`、かつ `ALLOWED_HOSTS` を具体的に。→ [settings.md](./settings.md)
- **`SECRET_KEY` をコミット**：セッション・CSRF・署名の鍵。漏れたら全署名が偽造可能。環境変数管理＋`.gitignore`。漏れたら**必ずローテーション**。→ [settings.md](./settings.md)
- **`|safe` / `mark_safe()` 乱用**：ユーザ入力に付けるとXSS直結。不安ならサニタイズ。
- **`raw()` / `extra()` での文字列連結**：`f"... '{q}'"` は即SQLインジェクション。必ず `%s` パラメータ。`.extra()` は非推奨気味なので原則ORM/`raw()`で。
- **`ALLOWED_HOSTS = ["*"]`**：Hostヘッダ汚染の穴。本番では具体ドメインを列挙。
- **オープンリダイレクト**：`redirect(request.GET["next"])` は外部誘導の穴。`url_has_allowed_host_and_scheme()` で検証。
- **CSRF を安易に `@csrf_exempt`**：解除するなら代替の認証（トークン等）を必ず用意する。

## 関連
[settings.md](./settings.md) / [templates.md](./templates.md) / [auth.md](./auth.md) / [middleware.md](./middleware.md) / [drf.md](./drf.md)
