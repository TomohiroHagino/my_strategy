# 認証・認可（Authentication / Authorization）（Django 5）

## ひとことで言うと
- **認証（Authentication）= 「あなたは誰か」** を確かめること（ログイン）。
- **認可（Authorization）= 「その操作をやってよいか」** を判断すること（権限）。
別物。Django では認証を `django.contrib.auth` が、認可を **permission / group** が担う。両方そろって初めて「正しいユーザーが、許された操作だけ」できる。

## 役割・なぜ必要か
- 認証だけでは「ログイン済みなら何でもできる」状態になり、**他人のリソースを操作できてしまう**（IDOR）。
- 認可は「このユーザーが、この対象に、この操作を許されているか」を1か所に集約する。
- Django は `User` モデル・セッション認証・permission を**標準装備**しているので、自前実装より標準を使うのが堅い。
- ただし **`AUTH_USER_MODEL`（カスタム User）はプロジェクト最初に決める**のが最重要。後から変えると地獄になる（後述）。

## 基本の書き方（コード）
### カスタム User（プロジェクト最初に設定する）
```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    # email ログインにしたい、フィールド追加したい等は最初にここで決める
    bio = models.TextField(blank=True)

# settings.py（最初の migrate より前に必ず）
AUTH_USER_MODEL = "accounts.User"
```
最小なら `AbstractUser`（username 等を引き継ぐ）、フルカスタムなら `AbstractBaseUser` + `BaseUserManager`。
**迷ったら `AbstractUser` で空サブクラスだけ作っておく**のが定石（後で拡張できる）。

### 認証：authenticate / login / logout
```python
# views.py
from django.contrib.auth import authenticate, login, logout

def login_view(request):
    user = authenticate(request, username=request.POST["username"],
                        password=request.POST["password"])
    if user is not None:          # 認証成功で User、失敗で None
        login(request, user)      # セッションにユーザーを保存＝ログイン状態
        return redirect("home")
    # 失敗時はエラー表示（ユーザー名/PWどちらが違うかは明かさない）

def logout_view(request):
    logout(request)               # セッション破棄
    return redirect("login")
```
実務では自前で書かず `django.contrib.auth.views.LoginView` / `LogoutView` を URL に繋ぐのが楽。

### ログイン必須にする（FBV / CBV）
```python
from django.contrib.auth.decorators import login_required
from django.contrib.auth.mixins import LoginRequiredMixin

@login_required                              # 関数ビュー
def dashboard(request): ...

class PostList(LoginRequiredMixin, ListView): # クラスビュー（Mixinは一番左に）
    model = Post
```
未ログインなら `settings.LOGIN_URL`（既定 `/accounts/login/`）へリダイレクトされる。

### Django 5.1：LoginRequiredMiddleware（全体ログイン必須）
```python
# settings.py — サイト全体をデフォルト「ログイン必須」にできる（5.1+）
MIDDLEWARE = [
    # ... AuthenticationMiddleware より後に置く
    "django.contrib.auth.middleware.LoginRequiredMiddleware",
]

# 公開したいビューだけ明示的に除外する
from django.contrib.auth.decorators import login_not_required

@login_not_required
def public_landing(request): ...
```
「基本は閉じる、開けるところだけ開ける」設計にしたい場合に便利。

### 認可：permission / group
```python
from django.contrib.auth.decorators import permission_required
from django.contrib.auth.mixins import PermissionRequiredMixin

# ビュー単位の権限チェック
@permission_required("blog.add_post", raise_exception=True)  # 無いと403
def create_post(request): ...

class PostUpdate(PermissionRequiredMixin, UpdateView):
    permission_required = "blog.change_post"

# コード内で直接判定
if request.user.has_perm("blog.delete_post"):
    ...
# group 単位の権限付与（管理画面 or コード）
editors = Group.objects.get(name="editors")
user.groups.add(editors)   # group の権限がそのユーザーに付く
```

### 認可の第一歩：そもそも他人のレコードを引かせない
```python
# URLのIDを差し替えても他人のデータに届かないようにスコープを絞る
post = get_object_or_404(Post, pk=pk, author=request.user)  # 自分の投稿のみ
# 一覧も同様に絞る
Post.objects.filter(author=request.user)
```
permission の有無に関わらず、**オブジェクトの所有者チェックは必須**。これが最も効く認可。

## 実務での使い方・定番パターン
- **`AUTH_USER_MODEL` は最初に確定**。空の `AbstractUser` サブクラスでもいいから初手で置く。
- 認証画面は `LoginView` / `LogoutView` / `PasswordChangeView` 等の**組み込みビュー**を URLconf に繋ぐ。
- ログイン必須は `LoginRequiredMixin`（CBV）/ `@login_required`（FBV）。5.1 なら `LoginRequiredMiddleware` で一括も可。
- **モデル標準 permission**（`add_/change_/delete_/view_<model>`）+ **独自 permission**（`Meta.permissions`）+ **group** で組む。
- 細かいオブジェクト単位の認可（「自分の投稿だけ編集」）は **所有者フィルタ**で。`django-guardian` 等もあるが、まずは filter で十分。
- パスワードハッシュは Django が自動（PBKDF2 既定）。`set_password()` を使い、平文を保存しない。

## ハマりどころ / アンチパターン
- **カスタム User を後から変更**（最頻・最重大）：マイグレーション済みプロジェクトで `AUTH_USER_MODEL` を変えるのは極めて困難。**プロジェクト最初に決める**。
- **認証だけして認可を忘れる**：ログインさえすれば `get_object_or_404(Post, pk=pk)` で他人の投稿を編集・削除できる（IDOR）。必ず `author=request.user` 等で絞る。
- **`LoginRequiredMixin` の位置ミス**：継承リストの**一番左**に置かないと効かない（`class V(LoginRequiredMixin, View)`）。
- **`@permission_required` で `raise_exception` 未指定**：既定はログイン画面へリダイレクトされ、403 にならず気付きにくい。API なら `raise_exception=True`。
- **テンプレートで非表示にしただけ＝認可ではない**：リンクを隠してもエンドポイントは叩ける。サーバ側で必ずチェック。
- **`is_staff` / `is_superuser` を権限代わりに乱用**：superuser は `has_perm` が常に True。本番ユーザーに安易に付けない。
- **`LoginRequiredMiddleware` を `AuthenticationMiddleware` より前に置く**：`request.user` が未設定で動かない。順序に注意。

## 関連
[middleware.md](./middleware.md) / [security.md](./security.md) / [settings.md](./settings.md)
