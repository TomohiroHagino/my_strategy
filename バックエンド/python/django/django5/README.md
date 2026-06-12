# Django 5 実務リファレンス（索引）

> **この版 = Django 5 系（5.0 / 5.1 / 5.2 LTS、Python 3.10+ 必須）**。各概念は1項目=1ファイルに切り出して詳しく書く。
> このREADMEは索引＋全体像だけ。詳細は各ファイルへ。

## この版のポイント（Django 5 で何が変わったか）
- **5.0**: admin の **facet フィルタ**、フォームの **field group** テンプレート、**`GeneratedField`**（DB計算カラム）、`db_default`、非同期サポート拡充。
- **5.1**: `{% querystring %}` テンプレートタグ、**`LoginRequiredMiddleware`**（全体ログイン必須を簡単に）、PostgreSQLコネクションプール。
- **5.2 LTS**: `shell` でのモデル自動import、**複合主キー（CompositePrimaryKey）**、`BoundField` のカスタマイズ。
- 全体として **async View / async ORM** が着実に強化中（ただし同期が依然主流）。

## 用語の注意（最初に）
Djangoは **MTV**。**「View」＝処理（他FWのコントローラ役）**、**「Template」＝見た目（他FWのビュー役）**。この呼び替えだけ先に頭に入れる。

## リクエストの流れ（全体像）
```
ブラウザ → urls.py(URLconf) → View（関数 or クラス）
        → Model(ORM)でDB操作 → Template でHTML生成 → レスポンス
（共通処理は Middleware が前後に挟まる）
```

## 項目（各ファイルへ）

### はじめに
- [getting_started.md](./getting_started.md) … プロジェクトの始め方（startproject / runserver / migrate）
- [project_apps.md](./project_apps.md) … project と app の構造とは

### コア（MTV）
- [request_flow.md](./request_flow.md) … リクエストの流れ・各層は何を返すか（全体俯瞰）
- [urls.md](./urls.md) … URLconf（ルーティング）とは
- [views.md](./views.md) … View（処理＝FBV / CBV）とは
- [templates.md](./templates.md) … テンプレート（DTL）とは
- [models.md](./models.md) … モデル（ORM）とは
- [orm_queries.md](./orm_queries.md) … クエリ（QuerySet / N+1）とは
- [migrations.md](./migrations.md) … マイグレーションとは
- [forms.md](./forms.md) … フォーム / ModelForm とは

### Django固有の強み
- [admin.md](./admin.md) … 管理画面（admin）とは
- [signals.md](./signals.md) … シグナルとは

### リクエスト処理・設定
- [middleware.md](./middleware.md) … ミドルウェアとは
- [auth.md](./auth.md) … 認証・認可とは
- [settings.md](./settings.md) … 設定（settings.py / .env）とは
- [static_media.md](./static_media.md) … 静的ファイル / メディアとは

### API・非同期・CLI
- [drf.md](./drf.md) … Django REST Framework（API）とは
- [celery.md](./celery.md) … Celery（非同期タスク）とは
- [management_commands.md](./management_commands.md) … manage.py コマンド / shell とは

### テスト・安全・運用
- [testing.md](./testing.md) … テスト（pytest-django）とは
  - [pytest.md](./pytest.md) … pytest / pytest-django（django_db・client・parametrize・conftest）とは
  - [factory_boy.md](./factory_boy.md) … factory_boy（DjangoModelFactory・SubFactory・Faker・Sequence）とは
- [security.md](./security.md) … セキュリティ（CSRF / XSS / SQLi）とは
- [pitfalls.md](./pitfalls.md) … 実務でハマる罠まとめ（早見表）

## 各ファイルの書式（テンプレ）
```markdown
# {概念名}（Django 5）
## ひとことで言うと
## 役割・なぜ必要か
## 基本の書き方（コード）
## 実務での使い方・定番パターン
## ハマりどころ / アンチパターン
## 関連: [xxx.md](./xxx.md)
```
