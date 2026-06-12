# 管理画面（admin）（Django 5）

## ひとことで言うと
モデルを登録するだけで、**追加・一覧・検索・編集・削除（CRUD）の管理UIが自動生成される**機能。`django.contrib.admin` が提供する、Django 最大の武器のひとつ。

## 役割・なぜ必要か
- 「DBの中身を人が見て直す」運用画面を、HTMLもViewも書かずに用意できる。社内の運用担当・管理者向けに即戦力。
- モデル定義（フィールド・関連・`__str__`）を元に、フォーム・バリデーション・一覧表をDjangoが組み立ててくれる。
- 認証・権限（`auth`）と統合済み。ログイン・パーミッション・操作ログ（`LogEntry`）まで標準装備。
- あくまで **運用者向け**であり、エンドユーザー向け画面ではない（そちらは通常のView/Templateで作る）。

## 基本の書き方（コード）
```python
# myapp/admin.py
from django.contrib import admin
from .models import Article, Comment


# 1) 最小：登録するだけでCRUD画面が出る
# admin.site.register(Article)

# 2) 実務はデコレータ + ModelAdmin で表示・操作をカスタムする
class CommentInline(admin.TabularInline):  # 親編集画面に子を埋め込む
    model = Comment
    extra = 1  # 空フォームを1行表示


@admin.register(Article)
class ArticleAdmin(admin.ModelAdmin):
    # 一覧に出す列（最初の列がリンクになる）
    list_display = ("title", "author", "status", "published_at")
    # 右サイドの絞り込み。Django 5 では facet 件数も出せる
    list_filter = ("status", "author")
    # 検索ボックスの対象。"関連__フィールド" 記法も可
    search_fields = ("title", "body", "author__username")
    # 編集不可で見せたい列（自動更新の日時など）
    readonly_fields = ("created_at", "updated_at")
    # 一覧から直接編集できる列
    list_editable = ("status",)
    # 日付ドリルダウンのナビ
    date_hierarchy = "published_at"
    # 親に子を埋め込む
    inlines = (CommentInline,)
    # slug を title から自動生成
    prepopulated_fields = {"slug": ("title",)}

    # 一覧クエリの N+1 を防ぐ（後述）
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        return qs.select_related("author")
```

```bash
# 管理者ユーザーを作る（これがないとログインできない）
python manage.py createsuperuser
# /admin/ にアクセス → ログイン
```

## 実務での使い方・定番パターン
- **`list_display` で一覧の情報量を上げる**。メソッドや関連も出せる。
  ```python
  @admin.register(Article)
  class ArticleAdmin(admin.ModelAdmin):
      list_display = ("title", "comment_count")

      @admin.display(description="コメント数", ordering="comment_count")
      def comment_count(self, obj):
          return obj.comment_count  # annotate した値を表示
  ```
- **Django 5 の facet フィルタ**：`list_filter` の各選択肢の横に件数を表示。`ArticleAdmin.show_facets` で制御（既定 `ShowFacets.ALLOW`、`?_facets` クエリで有効化）。常時出すなら `ShowFacets.ALWAYS`。
  ```python
  from django.contrib.admin import ShowFacets
  class ArticleAdmin(admin.ModelAdmin):
      show_facets = ShowFacets.ALWAYS  # 常に件数を表示
  ```
- **`fieldsets` で編集フォームをグループ化**し、長いフォームを整理。
  ```python
  fieldsets = (
      ("基本", {"fields": ("title", "slug", "author")}),
      ("公開", {"fields": ("status", "published_at"), "classes": ("collapse",)}),
  )
  ```
- **一括操作（actions）** で複数選択 → まとめて処理。
  ```python
  @admin.action(description="選択した記事を公開する")
  def make_published(modeladmin, request, queryset):
      queryset.update(status="published")  # 一括UPDATE（1クエリ）
  actions = (make_published,)
  ```
- **`__str__` を必ず定義**。一覧やドロップダウンの表示がこれに依存し、未定義だと `Article object (1)` になり運用が辛い。→ [models.md](./models.md)

## ハマりどころ / アンチパターン
- **本番で admin を無防備に晒すのが最大の危険**。`/admin/` のURLを推測されやすく、総当たりの標的になる。対策＝(1) URLを変える、(2) IP制限/VPN/Basic認証を前段に、(3) 管理者にも2要素認証、(4) `is_staff` 権限の付与を最小限に。→ [security.md](./security.md)
- **権限設定の理解不足**。`is_staff=True` で admin にログイン可、`is_superuser=True` で全権限。実務ではモデル単位のパーミッション（`add` / `change` / `delete` / `view`）を Group で割り当て、最小権限で運用する。→ [auth.md](./auth.md)
- **`list_display` での N+1**。関連フィールド（`author` など）を列に出すと、行ごとに追加クエリが飛ぶ。**`get_queryset()` で `select_related` / `prefetch_related`** を必ず入れる。`comment_count` のような集計は `annotate` で。→ [orm_queries.md](./orm_queries.md)
  ```python
  def get_queryset(self, request):
      from django.db.models import Count
      return (super().get_queryset(request)
              .select_related("author")
              .annotate(comment_count=Count("comment")))
  ```
- **`list_editable` と `list_display_links` の競合**。`list_editable` に入れた列はリンクにできない。先頭列を編集対象にするとエラーになる。
- **巨大テーブルで `list_filter` に高カーディナリティ列**を指定すると、フィルタ生成自体が重い。`raw_id_fields` や `autocomplete_fields` で外部キー選択を軽くする。
- **adminにビジネスロジックを書きすぎない**。`save_model` への過剰実装は隠れた副作用になる。ドメインロジックはモデル/サービスへ。→ [models.md](./models.md)

## 関連: [auth.md](./auth.md) / [models.md](./models.md) / [orm_queries.md](./orm_queries.md) / [security.md](./security.md)
