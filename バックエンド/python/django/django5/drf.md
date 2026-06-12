# Django REST Framework（API）（Django 5）

## ひとことで言うと
**DRF（Django REST Framework）= Djangoで JSON API を作るための定番サードパーティ製ライブラリ**（Django本体には含まれない）。`Serializer`（変換）・`View`/`ViewSet`（処理）・`Router`（URL生成）・認証/権限を組み合わせ、モデルを REST API として最短で公開できる。

## 役割・なぜ必要か
- Django本体は HTML を返す Web アプリ向け。**SPA / モバイル / 外部連携に JSON を返す**には、シリアライズ・認証・権限・ページネーションを毎回手書きするのは大変。
- DRF はそれらを **再利用可能な部品**として提供する。「モデル ↔ JSON の相互変換（Serializer）」「CRUDの定型処理（ViewSet）」「URL自動生成（Router）」「ブラウザで叩けるAPI画面（Browsable API）」が揃う。
- サードパーティなので `pip install djangorestframework` ＋ `INSTALLED_APPS` への `'rest_framework'` 追加が必須。

## 基本の書き方（コード）
```python
# settings.py
INSTALLED_APPS = [
    # ...
    "rest_framework",
]
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework.authentication.SessionAuthentication",
        "rest_framework.authentication.TokenAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",  # 既定で認証必須に
    ],
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```
```python
# serializers.py — モデル ↔ JSON の変換定義
from rest_framework import serializers
from .models import Author, Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ["id", "title", "published_at", "author"]  # 公開する項目を明示

class AuthorSerializer(serializers.ModelSerializer):
    books = BookSerializer(many=True, read_only=True)  # ネスト（関連を埋め込む）
    class Meta:
        model = Author
        fields = ["id", "name", "books"]
```
```python
# views.py — ViewSet で CRUD をまとめて定義
from rest_framework import viewsets, permissions
from .models import Author
from .serializers import AuthorSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    serializer_class = AuthorSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        # ★ N+1対策: 関連を事前取得（後述）
        return Author.objects.prefetch_related("books").all()
```
```python
# urls.py — Router が一覧/詳細URLを自動生成
from rest_framework.routers import DefaultRouter
from .views import AuthorViewSet

router = DefaultRouter()
router.register(r"authors", AuthorViewSet, basename="author")
urlpatterns = router.urls  # /authors/ , /authors/<pk>/ が自動で生える
```

## 実務での使い方・定番パターン
- **`ModelSerializer` を基本に**：モデルから自動でフィールド生成。`fields` は必ず明示列挙（`"__all__"` は項目漏れ・情報過剰露出の温床）。
- **`APIView` vs `ViewSet` の使い分け**：定型CRUDは `ModelViewSet`、独自ロジックが強いエンドポイントは `APIView`（`get`/`post` を自分で書く）。中間として `generics.ListCreateAPIView` 等もある。
- **Router**：`ViewSet` を `router.register` するだけで一覧/詳細/作成/更新/削除のURLが揃う。`APIView` は手書きの `path()` で登録する。
- **認証と権限は別物**：認証（誰か）＝`authentication_classes`、権限（許すか）＝`permission_classes`。`IsAuthenticated` / `IsAdminUser` / `IsAuthenticatedOrReadOnly` を使い分け、オブジェクト単位は `has_object_permission` を実装。→ [auth.md](./auth.md)
- **Browsable API**：開発中はブラウザで `/authors/` を開くとフォーム付きのUIで叩ける。本番では `DEFAULT_RENDERER_CLASSES` を `JSONRenderer` のみにして無効化することが多い。
- **ページネーション**：一覧は必ず `PAGE_SIZE` を設定。無制限の一覧返却はメモリ・転送量の事故になる。

## ハマりどころ / アンチパターン
- **シリアライザでN+1（最頻）**：ネストや関連参照を含むと、1件ごとに追加クエリが飛ぶ。**`get_queryset` で `select_related`（ForeignKey/OneToOne）/ `prefetch_related`（多対多・逆参照）を必ず付ける**。
  ```python
  # NG: 各 author ごとに books を都度クエリ（N+1）
  return Author.objects.all()
  # OK: 1〜2クエリにまとめる
  return Author.objects.prefetch_related("books").all()
  ```
  → 詳細は [orm_queries.md](./orm_queries.md)
- **権限設定漏れ**：`permission_classes` を付け忘れると `DEFAULT_PERMISSION_CLASSES` 次第で**誰でもアクセス可**に。既定を `IsAuthenticated` にし、公開APIだけ明示で緩める設計が安全。
- **ネストシリアライザの過剰クエリ・過剰ネスト**：深い入れ子は応答が肥大化しクエリも増える。一覧では軽い項目だけ、詳細でだけ展開する等、用途で Serializer を分ける。
- **`fields = "__all__"`**：パスワードハッシュや内部フラグまで漏らす危険。常に列挙する。
- **書き込み系のバリデーション欠如**：`create`/`update` は `serializer.is_valid(raise_exception=True)` を通す。入力検証は Serializer に集約する。→ [views.md](./views.md)
- **`many=True` のネストで書き込み**：ネストの作成/更新は自動では効かない。`create`/`update` を自分で実装する必要がある。

## 関連
[views.md](./views.md) / [orm_queries.md](./orm_queries.md) / [auth.md](./auth.md)
