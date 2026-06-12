# 実務でハマる罠まとめ（Pitfalls）（Django 5）

## ひとことで言うと
各ファイルの「ハマりどころ / アンチパターン」を集約した**早見表**。詳しくは各リンク先へ。レビュー前のセルフチェックや、原因切り分けの入口として使う。

## 役割・なぜ必要か
- Djangoは「便利な書き方」と「事故る書き方」が紙一重なものが多い。症状から該当箇所へ素早く飛ぶための索引。

## 用語 / MTV
- **MTV用語の取り違え**：Djangoの **「View」＝処理**（他FWのコントローラ役）、**「Template」＝見た目**。ここを混同すると会話も設計もズレる。→ [views.md](./views.md) / [templates.md](./templates.md)

## DB / ORM
- **N+1問題**：一覧で関連を1件ずつ引きSQLが `1+N` 回。FK/1対1は `select_related`、逆参照/多対多は `prefetch_related` でまとめ読み。検出は `django-debug-toolbar`。→ [orm_queries.md](./orm_queries.md)
- **`get()` の例外**：該当なしで `DoesNotExist`、複数該当で `MultipleObjectsReturned`。Web画面なら `get_object_or_404`、存在前提でないなら `filter().first()`。→ [orm_queries.md](./orm_queries.md)
- **`makemigrations` 忘れ**：モデルを変えてもマイグレーションを作らずDBと乖離。`makemigrations` → `migrate` をセットで。生成漏れは `makemigrations --check` で検出。→ [migrations.md](./migrations.md)
- **`bulk_create` / `update` / `QuerySet.update()` / `delete()`**：`save()`/シグナルやコールバックを経由しない場合がある。高速な代わりに整合性は自前で担保。→ [orm_queries.md](./orm_queries.md)
- **タイムゾーン**：`USE_TZ=True` 前提で `timezone.now()` を使う（`datetime.now()` は naive で事故る）。→ [settings.md](./settings.md)

## モデル / 設計
- **カスタムUserを後から差し替え**：`AUTH_USER_MODEL` はプロジェクト初期に決めるべき。運用後の差し替えは移行が非常に困難。最初に `AbstractUser` 継承で用意する。→ [auth.md](./auth.md)
- **fat view / fat model**：分岐・計算・複数モデルにまたがる手続きが膨らんだら、サービス層やフォーム/マネージャへ抽出。→ [views.md](./views.md) / [models.md](./models.md)
- **シグナルの暗黙副作用**：`post_save` 等にメール送信・別モデル更新を詰めると追跡不能・テスト困難に。明示的な呼び出しやサービス層を優先。→ [signals.md](./signals.md)

## セキュリティ / 設定
- **本番 `DEBUG = True`**：例外画面に `SECRET_KEY`・環境変数・SQLが露出。本番は `False`＋`ALLOWED_HOSTS` を具体指定。→ [settings.md](./settings.md) / [security.md](./security.md)
- **`SECRET_KEY` をコミット**：署名・セッション・CSRFの鍵。環境変数管理＋`.gitignore`、漏れたらローテーション。→ [settings.md](./settings.md)
- **`|safe` / `mark_safe()` でXSS**：ユーザ入力に付けると生HTML注入の穴。不安ならサニタイズ。→ [templates.md](./templates.md) / [security.md](./security.md)
- **`raw()`/`extra()` の文字列連結**：SQLインジェクション。`%s` パラメータで渡す。→ [security.md](./security.md)
- **CSRFトークン忘れ**：POSTフォームに `{% csrf_token %}` が無いと403。JS送信は `X-CSRFToken` ヘッダを付与。→ [security.md](./security.md)

## 運用 / デプロイ
- **本番 `collectstatic` 忘れ**：静的ファイルが配信されずCSS/JSが当たらない。デプロイ手順に `collectstatic` を必須化（`STATIC_ROOT` 設定前提）。→ [static_media.md](./static_media.md)
- **`manage.py check --deploy` 未実施**：Cookie secure・HSTS・`DEBUG` の不備を見逃す。CIに組み込む。→ [security.md](./security.md)

## 関連
[orm_queries.md](./orm_queries.md) / [migrations.md](./migrations.md) / [settings.md](./settings.md) / [security.md](./security.md) / [templates.md](./templates.md) / [auth.md](./auth.md) / [static_media.md](./static_media.md) / [signals.md](./signals.md)
