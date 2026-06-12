# リクエストの流れ・各層は何を返すか（Django 5）

## ひとことで言うと
1リクエストが **urls.py → Middleware → View → Model(ORM)** と降り、**Model が QuerySet/インスタンスを逆向きに上げてきて、View が Template(HTML) or JsonResponse に詰めて返す**。DjangoのViewは他FWのController役。DRFなら **View → Serializer → Model**。「どの層が何を受け取り、何を返すか」を1枚で俯瞰する。

## 全体の流れ（図）
```
ブラウザ
   │ リクエスト
   ▼
[urls.py(URLconf)]  URL→Viewを対応づけ
   │
   ▼
[Middleware]   認証・CSRF・セッションなど横断処理（前処理）
   │
   ▼
[View(views.py)]  入力を受け取る／Model(ORM) を呼ぶ ← Controller役
   │
   ▼
[Model(ORM)]   DBアクセス（QuerySet）
   │
   ▼
  DB ──→ QuerySet / インスタンス を返す ─┐
   ▲                                      │
[Model] が QuerySet/インスタンス を View に返す
   ▲
[View] が render(Template→HTML) or JsonResponse(JSON) に詰めて返す
   │
   ▼
[Middleware]   レスポンス後処理
   │ レスポンス（HTML or JSON）
   ▼
ブラウザ

（DRFの場合: View → Serializer（検証/シリアライズ）→ Model → Response）
```

## 各層は「何を受け取り・何を返す」か

| 層 | 受け取る | 返す |
|---|---|---|
| **urls.py** | URLパス | 対応するViewへ振り分け |
| **Middleware** | `HttpRequest` | request通過 or レスポンス（前後処理） |
| **View** | `HttpRequest` / URLパラメータ | **`HttpResponse`(render HTML)** or **`JsonResponse`** |
| **Model(ORM)** | 検索条件 / 保存データ | **QuerySet / モデルインスタンス** |
| **Serializer(DRF)** | リクエストdata / インスタンス | 検証済みdata / dict（JSON化前） |

- ViewはController役。入力を受け取り、Modelを呼び、Template or JsonResponseに詰めて返す。
- DRFでは Serializer が入力検証と出力整形の境界を担う（Laravelの FormRequest+Resource 相当）。

## コードで通して見る
```python
# 通常のDjango: View → Model → JsonResponse
from django.http import JsonResponse
from .models import Post

def create_post(request):
    post = Post.objects.create(           # ORMが Post インスタンスを返す
        title=request.POST["title"], body=request.POST["body"]
    )
    return JsonResponse({"id": post.id, "title": post.title})  # JSONに詰めて返す

# DRF: View → Serializer → Model → Response
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(["POST"])
def create_post_api(request):
    serializer = PostSerializer(data=request.data)  # 入力検証
    serializer.is_valid(raise_exception=True)        # NGなら400
    post = serializer.save()                         # ORMが Post を返す
    return Response(PostSerializer(post).data)        # Serializerで整形して返す
```

## 実務での使い方・定番パターン
- **層をまたいで返す型を決めておく**：Model→View は QuerySet/インスタンス、View→クライアントは HttpResponse(HTML) か JsonResponse/DRF Response。
- **API は DRF の Serializer で境界を固める**：入力検証も出力整形もSerializerに集約。→ [drf.md](./drf.md)
- **クエリは View で組み立てすぎない**：複雑な抽出は Model のマネージャ/メソッドへ。→ [orm_queries.md](./orm_queries.md)
- **横断処理は Middleware**：認証・ログ・CSRF はミドルウェアで共通化。→ [middleware.md](./middleware.md)

## ハマりどころ / アンチパターン
- **モデルインスタンスを `JsonResponse` に直接渡す**：シリアライズできない。DRF Serializer か dict に変換する。
- **N+1クエリ**：テンプレ/Serializerで関連を辿ると毎回クエリが飛ぶ。`select_related`/`prefetch_related`。→ [orm_queries.md](./orm_queries.md)
- **View に業務ロジックを盛りすぎ**：肥大化する。Model メソッドや Service に切り出す。→ [pitfalls.md](./pitfalls.md)
- **「View＝見た目」と勘違い**：DjangoのViewは処理（Controller役）。見た目はTemplate。→ [views.md](./views.md)

## 関連
[urls.md](./urls.md) / [views.md](./views.md) / [models.md](./models.md) / [templates.md](./templates.md) / [middleware.md](./middleware.md) / [drf.md](./drf.md)
