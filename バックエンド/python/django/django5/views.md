# View（処理＝FBV / CBV）（Django 5）

## ひとことで言うと
リクエストを受け取り、**モデルからデータを用意して Template か JSON を返す「処理の本体」**。Django の MTV では **View が他FWのコントローラ役**（見た目は Template）。`request` を受け取り `HttpResponse`（系）を返す callable が View。

## 役割・なぜ必要か
- URLconf（`urls.py`）から渡された `request` を起点に、**入力の受け取り（`request.GET` / `request.POST`）→ 認証/認可 → モデル操作 → 応答（`render` / `redirect` / `JsonResponse`）の決定**を担う層。
- 「処理の交通整理」だけを担当し、**業務ロジックは Model / Service（あるいは Form の `clean`）へ寄せて View を薄く**保つのが原則。判断や計算が増えた View は壊れやすく、テストもしにくい。
- View には2つの書き方がある。**FBV（関数ビュー）**＝素直で読みやすい。**CBV（クラスビュー、特に `ListView` / `DetailView` 等の generic）**＝定型 CRUD を継承で短く書ける。どちらも `request → HttpResponse` という契約は同じ。

## 基本の書き方（コード）
```python
# app/views.py
from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.views.generic import ListView, DetailView, CreateView
from django.urls import reverse_lazy
from .models import Post
from .forms import PostForm


# --- FBV（関数ビュー）: 一覧 ---
@login_required
def post_list(request):
    posts = Post.objects.filter(author=request.user).order_by("-created_at")
    # render(request, テンプレート名, context) が定番
    return render(request, "posts/list.html", {"posts": posts})


# --- FBV: 詳細（404を安全に返す）---
@login_required
def post_detail(request, pk):
    # 無ければ自動で 404。get() の DoesNotExist 例外を自前で握る必要がない
    post = get_object_or_404(Post, pk=pk, author=request.user)
    return render(request, "posts/detail.html", {"post": post})


# --- FBV: 作成（GETでフォーム表示 / POSTで保存）---
@login_required
def post_create(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user      # ログインユーザを所有者に
            post.save()
            return redirect("posts:detail", pk=post.pk)  # PRGパターン
    else:
        form = PostForm()
    return render(request, "posts/form.html", {"form": form})


# --- CBV（generic）: 同じ一覧/詳細/作成を短く ---
class PostListView(ListView):
    model = Post
    template_name = "posts/list.html"
    context_object_name = "posts"          # 既定は object_list

    def get_queryset(self):                # 自分の分だけに絞る
        return Post.objects.filter(author=self.request.user)


class PostDetailView(DetailView):
    model = Post                            # URLの <pk> で自動取得・404


class PostCreateView(CreateView):
    model = Post
    form_class = PostForm
    success_url = reverse_lazy("posts:list")  # 遅延評価でURL解決

    def form_valid(self, form):             # 保存直前にフックして所有者を入れる
        form.instance.author = self.request.user
        return super().form_valid(form)
```

## 実務での使い方・定番パターン
- **FBV と CBV の使い分け**：単純で素直なものや独自処理が多いものは FBV。**定型 CRUD は generic CBV**（`ListView` / `DetailView` / `CreateView` / `UpdateView` / `DeleteView`）で短く。チームの統一方針を先に決めると迷わない。
- **`render(request, template, context)`** が表示の基本。`context` は `dict`。`redirect("name", pk=...)` は URL名で安全に遷移。
- **PRG（Post / Redirect / Get）**：POST 成功後は必ず `redirect`。リロード時の二重送信を防ぐ。
- **`get_object_or_404`** で「無ければ 404」を1行に。`Post.objects.get()` の `DoesNotExist` を自前 try/except するより安全・簡潔。
- **認可は QuerySet で自然に**：`filter(author=request.user)` / `get_object_or_404(Post, pk=pk, author=request.user)` のように常にログインユーザのスコープで引くと、他人のリソース操作を構造的に防げる。
- **CBV のメソッドを理解する**：`get()` / `post()` がHTTPメソッドに対応し、`get_queryset()`（対象の絞り込み）、`get_context_data()`（context追加）、`form_valid()`（保存フック）を override して挙動を調整するのが定石。
- **ログイン必須**：FBV は `@login_required`、CBV は `LoginRequiredMixin` を最左に継承。5.1 以降は `LoginRequiredMiddleware` で全体を一括必須化もできる。→ [auth.md](./auth.md)

## ハマりどころ / アンチパターン
- **Fat View（処理の詰め込み）**：分岐・集計・通知などを View に書き続けると太る。**ロジックは Model のメソッド / Service / Form の `clean` へ**逃がす。
- **CBV のメソッドを理解せず override**：`get_queryset` と `get_context_data` を取り違える、`form_valid` で `super()` を呼ばず保存されない、`success_url` に `reverse`（即時評価）を書いてインポート時に落ちる（→ `reverse_lazy` を使う）。
- **`get()` で `DoesNotExist` 未処理**：500 になる。`get_object_or_404` を使う。
- **認可忘れ**：`Post.objects.get(pk=pk)` だと他人のレコードも引ける。必ずユーザでスコープする。
- **POST 後に `render` で再表示**：リロードで二重送信になる。成功時は `redirect`（PRG）。
- **`request.POST` を生で信頼**：バリデーションせず保存しない。**Form / ModelForm を通す**。→ [forms.md](./forms.md)
- **テンプレート内で関連を辿って N+1**：View 側で `select_related` / `prefetch_related` して渡す。→ [orm_queries.md](./orm_queries.md)

## 関連
[urls.md](./urls.md) / [templates.md](./templates.md) / [models.md](./models.md) / [forms.md](./forms.md) / [auth.md](./auth.md)
