# クエリ（QuerySet / N+1）（Django 5）

## ひとことで言うと
**QuerySet** は「DBへの問い合わせ条件をためた、まだ実行していないオブジェクト」。`Article.objects.filter(...)` のように組み立て、**実際に値が必要になった瞬間に1回だけSQLが走る**（遅延評価 / lazy）。Railsの ActiveRecord リレーション相当。

## 役割・なぜ必要か
- DBアクセスをPythonのメソッド呼び出しに翻訳し、生SQLの重複・SQLインジェクション・タイプミスを避けるためにある。
- **遅延評価**のおかげで `filter().exclude().order_by()` とチェーンしても、その都度SQLが走らず最後にまとめて1クエリになる。だが裏返すと「いつ評価されるか」を理解しないとN+1や無駄クエリを生む。

## 基本の書き方（コード）
```python
from django.db.models import Q, F, Count, Sum
from django.db import transaction

# --- 取得系 ---
Article.objects.all()                          # 全件（まだSQLは走らない）
Article.objects.filter(status="published")     # 条件で絞る
Article.objects.exclude(status="draft")        # 条件で除外
Article.objects.filter(view_count__gte=100)    # __gte/__lte/__contains/__in などのlookup
Article.objects.get(pk=1)                       # 1件取得（0件/複数件は例外）
Article.objects.filter(...).first()            # 0件なら None（例外を出したくない時）
Article.objects.order_by("-published_at")[:10] # 並べて先頭10件（LIMIT）

# --- 評価が走るタイミング ---
qs = Article.objects.filter(status="published")  # ここではまだSQLなし
for a in qs: ...        # イテレートで評価（SQL実行・結果キャッシュ）
list(qs); len(qs); bool(qs)  # これらも評価のトリガー

# --- 集計 ---
Article.objects.aggregate(total=Sum("view_count"))         # 全体集計 → dict
Article.objects.values("author").annotate(n=Count("id"))   # GROUP BY 集計

# --- F() / Q() ---
Article.objects.filter(pk=1).update(view_count=F("view_count") + 1)  # DB内で加算（競合に強い）
Article.objects.filter(Q(title__icontains="django") | Q(body__icontains="django"))  # OR条件

# --- 軽量取得 ---
Article.objects.values("id", "title")        # dictのQuerySet（必要列だけ）
Article.objects.values_list("id", flat=True) # [1, 2, 3] のように列だけ
```

## 実務での使い方・定番パターン
- **N+1 を `select_related` / `prefetch_related` で潰す**（最重要）:
  ```python
  # FK・OneToOne（多対1/1対1）は select_related → JOINで1クエリ
  for a in Article.objects.select_related("author"):
      print(a.author.name)        # 追加クエリなし

  # M2M・逆参照（多対多/1対多）は prefetch_related → 別クエリでまとめ取り
  for a in Article.objects.prefetch_related("categories"):
      print([c.name for c in a.categories.all()])  # N回ではなく2クエリ

  # 絞り込んだ/別属性に格納する prefetch は Prefetch() オブジェクト
  from django.db.models import Prefetch
  Article.objects.prefetch_related(
      Prefetch("comments",
               queryset=Comment.objects.filter(approved=True),
               to_attr="approved_comments")   # a.approved_comments で参照
  )
  ```
  > 使い分け: **FK/OneToOne → `select_related`（JOIN）**、**M2M/逆参照 → `prefetch_related`（別クエリIN取得）**。条件付き・別名は `Prefetch`。
- **トランザクション**: 複数の書き込みをまとめて、途中失敗で全部巻き戻す。
  ```python
  with transaction.atomic():
      order.save()
      stock.quantity = F("quantity") - 1
      stock.save()          # どれか失敗で全部ロールバック
  ```
- **一括作成 / 更新**: ループ保存は遅い。`bulk_create` / `bulk_update` でまとめる。
  ```python
  Article.objects.bulk_create([Article(title=f"t{i}", author=u) for i in range(1000)])
  ```
- **大量データはイテレータで**: `Article.objects.all().iterator(chunk_size=2000)` でメモリを抑える。
- **存在確認は `.exists()`**、件数は `.count()`。`len(qs)` や `if qs:` は全件ロードするので避ける。
- **`get_or_create` / `update_or_create`** で「無ければ作る」を1メソッドで。
- カスタムマネージャ/QuerySetで `Article.objects.published()` のように業務クエリへ名前を付けると再利用しやすい。

## ハマりどころ / アンチパターン
- **N+1問題（最頻）**: 一覧で `a.author.name` を回すと「1（一覧）＋N（各著者）」回SQLが走る。FK/OneToOneは `select_related`、M2M/逆参照は `prefetch_related` で解消。検出は **django-debug-toolbar** や **nplusone** が定番。
- **`get()` の例外**: 該当0件で `DoesNotExist`、複数件で `MultipleObjectsReturned`。Webでは `get_object_or_404()` を使うか、`filter().first()` で `None` 判定する。
- **遅延評価の誤解**: QuerySetは評価のたびにSQLを再発行する。同じ結果を何度も使うなら `qs = list(...)` で一度マテリアライズしてキャッシュする。逆に、評価済みQuerySetに `.filter()` を足すと新クエリになる点にも注意。
- **`update()` はシグナル/`save()`を通さない**: `QuerySet.update()` や `bulk_create` は `save()`・`pre_save`シグナル・`auto_now` を発火しない。フックが必要なら個別 `save()`。
- **スライス後にfilter不可**: `qs[:10].filter(...)` はエラー。絞り込みはスライス前に。
- **N件取得で `F()` を使わずカウンタ加算**: `obj.view_count += 1; obj.save()` は読み→書きの間に競合する。`update(view_count=F("view_count") + 1)` でDB内加算する。
- **`null` を含む `exclude`**: SQLのNULL三値論理で直感と違う結果になることがある。NULLは `isnull=True` で明示的に扱う。

## さらに知っておく現象（性能・整合）
| 現象 / 罠 | なぜ | 対策 |
|---|---|---|
| 不要な全カラム取得 | 既定で全列をロードしモデル生成 | **`only("id","title")` / `defer("body")`**、表示だけなら **`values()` / `values_list()`**（辞書/タプルで軽量） |
| Python ループで集計 | `sum(a.likes for a in qs)` は全件ロード | **`aggregate(Sum("likes"))`**、グループ別は **`annotate(Count("comments"))`** でDB集計 |
| 競合する read-modify-write | `obj.n += 1; obj.save()` は読み書きの間に競合 | **`F("n") + 1`** でDB内更新。在庫減算等は `select_for_update()`（行ロック・`transaction.atomic` 内） |
| QuerySet 再評価でクエリ増 | 遅延評価＋結果キャッシュ。別変数でスライス/再イテレートすると再クエリ | 使い回すなら `qs = list(qs)` で一度マテリアライズ |
| 接続の張り直し | リクエスト毎に接続を開閉すると負荷時に重い | `DATABASES` の **`CONN_MAX_AGE`**（永続接続）を設定。プーラ併用も検討 |

## 関連
[models.md](./models.md) / [migrations.md](./migrations.md) / [security.md](./security.md)
