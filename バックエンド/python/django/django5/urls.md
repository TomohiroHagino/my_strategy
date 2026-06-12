# URLconf（ルーティング）（Django 5）

## ひとことで言うと
**「どのURLに、どのView（処理）を割り当てるか」を定義する対応表**。`urls.py` の `urlpatterns` に `path()` を並べて書く。MTV の入口で、リクエストを View（＝コントローラ役の処理）へ振り分ける。

## 役割・なぜ必要か
- リクエストは `mysite/urls.py`（ルートのURLconf）から照合され、**上から順にマッチした最初の `path()`** の View が呼ばれる。
- URL と View を分離することで、URL設計を変えても View 本体は触らずに済む（疎結合）。
- `<int:pk>` のような**パスコンバータ**でURL中の値を取り出し、View に引数として渡せる（型変換つき）。
- `name=` を付けておくと、テンプレートやコードから **URLを文字列直書きせず逆引き**できる（`{% url %}` / `reverse()`）。URLを後で変えても参照側が壊れない。

## 基本の書き方（コード）
```python
# mysite/urls.py … ルート（project側）
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("blog/", include("blog.urls")),   # app の urls.py を取り込む
    path("", include("blog.urls")),        # トップも blog に委譲する例
]
```

```python
# blog/urls.py … app側（app_name で名前空間を切る）
from django.urls import path, re_path
from . import views

app_name = "blog"                          # 名前空間（逆引きで blog:detail と書ける）

urlpatterns = [
    path("", views.index, name="index"),
    path("articles/<int:pk>/", views.detail, name="detail"),
    path("tag/<slug:slug>/", views.by_tag, name="by_tag"),
    # 正規表現が必要なときだけ re_path（基本は path で足りる）
    re_path(r"^archive/(?P<year>[0-9]{4})/$", views.archive, name="archive"),
]
```

```python
# blog/views.py … pk はコンバータ <int:pk> で int として渡る
from django.shortcuts import render, get_object_or_404, redirect
from django.urls import reverse
from .models import Article


def detail(request, pk):
    article = get_object_or_404(Article, pk=pk)
    return render(request, "blog/detail.html", {"article": article})


def go_top(request):
    # Pythonコードからの逆引き（名前空間つき）
    return redirect(reverse("blog:index"))
```

```html
<!-- テンプレートからの逆引き（URLを直書きしない） -->
<a href="{% url 'blog:detail' pk=article.pk %}">{{ article.title }}</a>
<a href="{% url 'blog:index' %}">一覧へ</a>
```

主なパスコンバータ: `str`（既定・/を含まない文字列）/ `int`（整数）/ `slug`（英数とハイフン）/ `uuid` / `path`（/を含む全体）。

## 実務での使い方・定番パターン
- **app ごとに `urls.py` を持たせ、project の `urls.py` から `include()`** で束ねる。URLもモジュール化でき、app の再利用が効く。→ [project_apps.md](./project_apps.md)
- **必ず `name=` を付け、参照は逆引き**（`{% url %}` / `reverse()` / `redirect("blog:detail", pk=...)`）。URL直書きは後で全部壊れる元。
- **`app_name` で名前空間**を切り、`blog:detail` のように呼ぶ。app をまたいで同名 `detail` があっても衝突しない。
- **CBV は `.as_view()` を path に渡す**：`path("articles/<int:pk>/", views.ArticleDetail.as_view(), name="detail")`。→ [views.md](./views.md)
- 末尾スラッシュは**「ディレクトリ風URLは末尾 `/` 付き」で統一**するのが Django 流（既定の `APPEND_SLASH=True` と相性が良い）。
- 正規表現が要るときだけ `re_path`。基本は `path` のコンバータで足りる（可読性が高い）。

## ハマりどころ / アンチパターン
- **末尾スラッシュ（APPEND_SLASH）**：`path("articles/")` に対し `/articles`（スラッシュ無し）で来ると、GET は自動で `/articles/` へリダイレクトされるが、**POST はリダイレクトされず 404 になりがち**。フォーム action やAjaxのURLは末尾 `/` を正確に合わせる。
- **`name` の取り違え／名前空間漏れ**：`{% url 'detail' %}` と書いて `NoReverseMatch`。`app_name` を切ったら `blog:detail` と**名前空間付き**で呼ぶ。引数（`pk=` など）の過不足も `NoReverseMatch` の原因。
- **include の順序**：`urlpatterns` は**上から最初にマッチした方**が勝つ。広く一致する `path("", ...)` や `re_path` を上に置くと、後続の具体的ルートに届かない。**具体的→汎用の順**で並べる。
- **`include` の二重スラッシュ**：`path("blog/", include(...))` の中で `path("/x/", ...)` のように先頭 `/` を付けると `blog//x/` になる。app 側は**先頭スラッシュ無し**で書く。
- **正規表現の濫用**：`path` で済むものを `re_path` で書くと読みにくくバグの温床。コンバータ優先。
- **View にURL都合を持ち込む**：URLの整形・判定は URLconf 側に寄せ、View は処理に集中させる。

## 関連
[views.md](./views.md)
