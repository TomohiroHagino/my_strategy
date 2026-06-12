# フォーム / ModelForm（Django 5）

## ひとことで言うと
ユーザー入力の **「受け取り・検証・整形」を一手に引き受ける層**。`forms.Form`（任意の入力用）と `forms.ModelForm`（モデルに直結する入力用）があり、**Rails の Strong Parameters ＋ バリデーション**に相当する。

## 役割・なぜ必要か
- HTML フォームから来る生の `request.POST` は信頼できない外部入力。Form は **どのフィールドを受け取るか（許可リスト）・型変換・検証**をまとめて担当し、安全に通った値だけを `cleaned_data` として渡す。
- **検証ロジックを View から切り出せる**。View は薄く保ち、入力の正しさは Form に集約できる（Fat View 回避）。
- **テンプレートへの描画も Form が持つ**（`{{ form.as_p }}` 等）。ラベル・入力欄・エラー表示を自動生成でき、HTML を手書きする量が減る。
- `ModelForm` なら **モデル定義からフィールドを自動生成**し、`form.save()` でそのまま DB 保存までできる。

## 基本の書き方（コード）
```python
# app/forms.py
from django import forms
from .models import Post


# --- forms.Form : モデルに紐づかない入力（検索・問い合わせ等）---
class ContactForm(forms.Form):
    name = forms.CharField(max_length=50)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)   # ウィジェットで見た目指定

    # フィールド単位の検証（clean_<フィールド名>）
    def clean_email(self):
        email = self.cleaned_data["email"]
        if email.endswith("@example.com"):
            raise forms.ValidationError("そのドメインは使えません")
        return email                                   # 必ず値を return


# --- forms.ModelForm : モデル直結（CRUD の定番）---
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "body", "is_public"]   # 受け取るフィールドを明示（重要）
        widgets = {
            "body": forms.Textarea(attrs={"rows": 6}),
        }

    # 複数フィールドにまたがる検証は clean()
    def clean(self):
        cleaned = super().clean()
        if cleaned.get("is_public") and not cleaned.get("body"):
            raise forms.ValidationError("公開するには本文が必要です")
        return cleaned
```

```python
# app/views.py（FBV での定番フロー）
from django.shortcuts import render, redirect
from .forms import PostForm

def post_create(request):
    if request.method == "POST":
        form = PostForm(request.POST)
        if form.is_valid():                # ここで初めて検証が走る
            post = form.save(commit=False)  # まだ保存しない
            post.author = request.user     # 追加情報を埋めて
            post.save()                    # 保存
            return redirect("posts:detail", pk=post.pk)
    else:
        form = PostForm()                  # GET は空フォーム
    return render(request, "posts/form.html", {"form": form})
```

```html
{# templates/posts/form.html : CSRF は必須 #}
<form method="post">
  {% csrf_token %}
  {{ form.as_p }}        {# ラベル・入力欄・エラーをまとめて描画 #}
  <button type="submit">保存</button>
</form>
```

## 実務での使い方・定番パターン
- **`is_valid()` → `cleaned_data` の順**：`form.is_valid()` を呼んで初めて検証が実行され、`cleaned_data`（型変換・検証済みの値）が埋まる。これを呼ぶ前に `cleaned_data` を触らない。
- **検証は `clean_<field>()` と `clean()` で書く**：1フィールドの検証は `clean_<field>`（戻り値で値を返す）、複数フィールドにまたがる検証は `clean()`。エラーは `ValidationError` を投げる。
- **`ModelForm` の `save(commit=False)`**：所有者やタイムスタンプなど画面外の値を入れたいときは、保存を保留して属性を埋めてから `save()`。
- **ウィジェット**で見た目・属性を制御（`Textarea`、`attrs={"class": "...", "placeholder": "..."}`、`DateInput(type="date")` 等）。
- **テンプレート描画**：手早くは `{{ form.as_p }}` / `{{ form.as_div }}`（5系既定）。細かく組むなら `{{ form.title.label_tag }}{{ form.title }}{{ form.title.errors }}` とフィールド単位で。
- **検索やフィルタ**はモデルに紐づかないので `forms.Form` を使い、`cleaned_data` を QuerySet の `filter` に渡す。

## ハマりどころ / アンチパターン
- **`is_valid()` 前に `cleaned_data` へアクセス**：`AttributeError` や空挙動になる。必ず `is_valid()` を通してから。
- **`Meta.fields` の指定漏れ / `fields = "__all__"`**：`__all__` や未指定だと**意図しないフィールドまで受け付け**、ユーザーが本来編集できない項目（権限・所有者・公開フラグ等）を上書きできる **mass assignment 的な穴**になる。**受け取りたいフィールドを明示的に列挙**する。
- **サーバ側検証を省く**：HTML5 の `required` や JS バリデーションはバイパス可能。**検証は必ずサーバ側（Form）で**行う。クライアント検証は補助。
- **`clean_<field>()` で値を return し忘れる**：戻り値が `cleaned_data` に反映されないため、後段で値が消える。検証後は必ず値を return。
- **`{% csrf_token %}` の付け忘れ**：POST が 403 になる。→ [templates.md](./templates.md)
- **エラー表示を描画しない**：`{{ form.as_p }}` を使わず手組みした場合、`{{ form.errors }}` / フィールドの `.errors` を出さないとユーザーに失敗理由が伝わらない。

## 関連
[views.md](./views.md) / [models.md](./models.md) / [templates.md](./templates.md) / [security.md](./security.md)
