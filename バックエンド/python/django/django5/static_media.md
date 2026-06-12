# 静的ファイル / メディア（Static / Media）（Django 5）

## ひとことで言うと
- **静的ファイル（static）= 開発者が用意する固定資産**（CSS / JS / ロゴ画像など）。コードと一緒に Git 管理する。
- **メディア（media）= ユーザーがアップロードする可変ファイル**（プロフィール画像・添付など）。Git 管理しない。
この2つは**別物**で、配信方法も保存場所も分ける。

## 役割・なぜ必要か
- Django 本体は HTML を返すのが仕事で、**静的ファイル配信は本来 Web サーバ（nginx 等）や CDN の役目**。開発時だけ Django が肩代わりする。
- 静的ファイルは複数アプリに散らばるので、本番前に**1か所へ集約（collectstatic）**してから配信する必要がある。
- メディアは「ユーザー入力」なので、配信先・サイズ・拡張子の扱いを静的とは別に管理する必要がある（セキュリティ）。

## 基本の書き方（コード）
### 静的ファイルの設定
```python
# settings.py
STATIC_URL = "static/"                       # URL 上の参照プレフィックス → /static/...
STATICFILES_DIRS = [BASE_DIR / "assets"]     # collectstatic が集める「元」の追加場所
STATIC_ROOT = BASE_DIR / "staticfiles"       # collectstatic の「集約先」（本番配信元）
# 各アプリの app/static/<app>/ も自動で収集対象になる
```
```django
{# テンプレート側 #}
{% load static %}
<link rel="stylesheet" href="{% static 'css/app.css' %}">
<img src="{% static 'img/logo.png' %}">
```
```bash
# 本番デプロイ前に必ず実行：散らばった static を STATIC_ROOT に集約
python manage.py collectstatic --no-input
```

### メディア（ユーザーアップロード）の設定
```python
# settings.py
MEDIA_URL = "media/"                 # URL プレフィックス → /media/...
MEDIA_ROOT = BASE_DIR / "media"      # 実ファイルの保存ディレクトリ
```
```python
# models.py — アップロードを受けるフィールド
class Profile(models.Model):
    avatar = models.ImageField(upload_to="avatars/%Y/%m/")  # MEDIA_ROOT 配下に保存
    # ImageField は Pillow が必要：pip install Pillow
```
```python
# urls.py — 開発サーバでだけ media を配信（本番ではやらない）
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [ ... ]
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### 本番配信パターン1：WhiteNoise（static を Django/Gunicorn から配る）
```python
# pip install whitenoise
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",   # SecurityMiddleware の直後
    # ...
]
# ハッシュ付きファイル名＋圧縮（キャッシュに強い）
STORAGES = {
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}
```
WhiteNoise は **static 専用**。media は対象外なので S3 等を使う。

### 本番配信パターン2：nginx（Django の前段で直接返す）
```nginx
location /static/ { alias /app/staticfiles/; }   # STATIC_ROOT を指す
location /media/  { alias /app/media/; }          # MEDIA_ROOT を指す
location /        { proxy_pass http://127.0.0.1:8000; }  # 動的はDjangoへ
```

### 本番配信パターン3：S3 など外部ストレージ（django-storages）
```python
# pip install django-storages boto3
STORAGES = {
    "default":     {"BACKEND": "storages.backends.s3.S3Storage"},  # media を S3 へ
    "staticfiles": {"BACKEND": "storages.backends.s3.S3Storage"},
}
AWS_STORAGE_BUCKET_NAME = env("AWS_STORAGE_BUCKET_NAME")  # 鍵は環境変数で
```
スケールするなら media は S3、配信は CloudFront 等の CDN が定番。

## 実務での使い方・定番パターン
- **static と media を最初から分けて考える**。「固定資産＝static / ユーザー入力＝media」を混同しない。
- 小〜中規模は **WhiteNoise（static）＋ S3 or ローカル（media）**、規模が出たら **nginx / CDN / S3** に寄せる。
- デプロイ手順に **`collectstatic` を必ず組み込む**（CI/CD やコンテナ起動時に自動実行）。
- 5.0+ の **`STORAGES` 設定**（`default` / `staticfiles`）に統一。旧 `DEFAULT_FILE_STORAGE` / `STATICFILES_STORAGE` は非推奨。
- アップロードは**拡張子・サイズ・MIME を検証**し、`upload_to` で日付ディレクトリに分散して1ディレクトリ肥大を防ぐ。

## ハマりどころ / アンチパターン
- **本番で `collectstatic` 忘れ**（最頻）：CSS/JS が 404 になり「本番だけ見た目が崩れる」。デプロイ手順に必ず入れる。
- **`DEBUG=False` で開発サーバが static を返さない**：`runserver` の自動配信は `DEBUG=True` のときだけ。本番相当で確認するなら WhiteNoise 等を入れる。
- **static と media の混同**：ユーザーアップロードを `STATICFILES_DIRS` に置く／`collectstatic` 先に置く、はNG。可変ファイルは `MEDIA_ROOT`。
- **`STATIC_ROOT` を `STATICFILES_DIRS` と同じにする**：`collectstatic` が自分の収集先を元として食って事故る。別ディレクトリにする。
- **本番で `static()`（urls.py の DEBUG 配信）に依存**：あれは開発専用。本番は Web サーバ／CDN／WhiteNoise で配る。
- **media を公開ディレクトリに直置きして権限チェックなし**：非公開ファイルは認可付きビュー経由で返す（直リンクで漏れる）。
- **`ImageField` で Pillow 未インストール**：マイグレーション or 保存時にエラー。`pip install Pillow` を忘れない。

## 関連
[settings.md](./settings.md) / [security.md](./security.md) / [getting_started.md](./getting_started.md)
