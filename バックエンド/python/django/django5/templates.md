# テンプレート（DTL）（Django 5）

## ひとことで言うと
ユーザーに見せる **HTMLを組み立てる「見た目」の層**。Django の MTV では **Template が他FWのビュー役**。標準は **DTL（Django Template Language）**で、`{{ 変数 }}` で値を出力、`{% タグ %}` でロジックを書く。

## 役割・なぜ必要か
- MTV の「見た目」担当。View が `render(request, "...", context)` で渡した **context（dict）の値を埋め込んで HTML を生成**する。
- DTL は意図的に機能を絞った言語で、**テンプレート内に複雑なロジックを書けないようになっている**。表示に専念させ、計算や判断は View / Model 側に置く設計思想。
- 最大の安全装置が **自動エスケープ（autoescape）**。`{{ 変数 }}` の出力は既定で `<`, `>`, `&`, `"` 等を HTML エスケープし、**XSS を自動で防ぐ**。だからこそ `|safe` でこれを外すのは危険。

## 基本の書き方（コード）
```html
{# templates/posts/list.html #}
{% extends "base.html" %}   {# 親テンプレートを継承 #}

{% block content %}         {# base.html の同名 block を差し替え #}
  <h1>{{ title }}</h1>

  {% if posts %}
    <ul>
      {% for post in posts %}
        {# {% url %} で URL名から逆引き（パスのハードコード回避） #}
        <li>
          <a href="{% url 'posts:detail' post.pk %}">{{ post.title }}</a>
          <small>{{ post.created_at|date:"Y/m/d H:i" }}</small>   {# フィルタ #}
        </li>
      {% empty %}            {# for が空のときだけ表示 #}
        <li>まだ投稿がありません</li>
      {% endfor %}
    </ul>
  {% else %}
    <p>データがありません</p>
  {% endif %}

  {% include "posts/_summary.html" with count=posts.count %}  {# 部品の差し込み #}
{% endblock %}
```

```html
{# templates/base.html : 全ページ共通の枠 #}
<!doctype html>
<html lang="ja">
<head><title>{% block title %}サイト{% endblock %}</title></head>
<body>
  <header>{% include "shared/_nav.html" %}</header>
  <main>{% block content %}{% endblock %}</main>  {# 子がここを埋める #}
</body>
</html>
```

```html
{# フォーム：POST には CSRF トークンが必須 #}
<form method="post" action="{% url 'posts:create' %}">
  {% csrf_token %}              {# これが無いと 403 Forbidden #}
  {{ form.as_p }}
  <button type="submit">保存</button>
</form>
```

## 実務での使い方・定番パターン
- **テンプレート継承で枠を共有**：`base.html` に共通レイアウト（ヘッダ/フッタ）と `{% block %}` を置き、各ページは `{% extends %}` して必要な block だけ差し替える。これが基本構造。
- **`{% include %}` で部品化**：カードやナビなど繰り返す断片を `_xxx.html` に切り出し、`with` でローカル変数を渡す。Rails の partial に相当。
- **`{% url 'name' arg %}` で URL を逆引き**：パスを直書きせず URL名で参照。ルート変更に強い。
- **フィルタで整形**：`{{ value|date:"Y/m/d" }}`、`{{ text|truncatechars:50 }}`、`{{ qty|default:"0" }}`、`{{ name|upper }}`。表示用の軽い加工はフィルタで。
- **`{% csrf_token %}` は POST フォームに必須**。Django の CSRF 保護が前提（→ [forms.md](./forms.md) / [security.md](./security.md)）。
- **context の渡し方**：View で `render(request, "...", {"posts": posts})` のように dict で渡す。テンプレートではドットで属性・辞書キー・メソッド（引数なし）を辿れる（`post.author.name`）。
- **5系の新機能**：5.1 の `{% querystring %}` で現在のクエリ文字列を保ったままページ送りリンクが作れる。フォームは field group テンプレートで描画をカスタムできる。

## ハマりどころ / アンチパターン
- **`|safe` / `{% autoescape off %}` の乱用 → XSS**：ユーザー入力に `|safe` を付けると自動エスケープが外れ、`<script>` がそのまま実行され得る。**信頼できない値に `|safe` は絶対に付けない**。HTML を通したいときは `bleach` 等でサニタイズしてから。→ [security.md](./security.md)
- **テンプレートにロジックを書きすぎる**：分岐や集計を `{% if %}` だらけで書くと読めない。**判断・計算は View / Model へ**逃がし、テンプレートは表示に専念させる。
- **テンプレート内アクセスで N+1**：`{% for p in posts %}{{ p.author.name }}{% endfor %}` は author を毎回 DB から引く。View 側で `select_related("author")` / `prefetch_related(...)` してから渡す。→ [orm_queries.md](./orm_queries.md)
- **`{% csrf_token %}` の付け忘れ**：POST フォームで 403。GET フォームには不要。
- **存在しない変数の参照**：DTL は黙って空文字になる（例外を投げない）ので、誤字やキー名ミスに気づきにくい。`{% if %}` で存在チェックする癖を。
- **テンプレートで関数を引数付き呼び出ししようとする**：DTL は引数付きメソッド呼び出し不可。必要なら View で計算するかカスタムフィルタ/タグを作る。

## 関連
[views.md](./views.md) / [forms.md](./forms.md) / [urls.md](./urls.md) / [security.md](./security.md) / [orm_queries.md](./orm_queries.md)
