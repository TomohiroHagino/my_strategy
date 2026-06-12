# 認証・認可（Authentication / Authorization）（Laravel 11）

## ひとことで言うと
- **認証（Authentication）=「あなたは誰か」** を確かめること（ログイン）。Laravel ではスターターキット（Breeze/Jetstream）・Fortify・**Sanctum**（SPA/APIトークン）・Passport（OAuth）が担う。
- **認可（Authorization）=「その操作をやってよいか」** を判断すること（権限）。Laravel では **Gate** と **Policy** が担う。
別物で、両方そろって初めて「正しいユーザーが、許された操作だけ」できる。

## 役割・なぜ必要か
- 認証だけだと「ログイン済みなら何でもできる」状態になり、**URLのIDを差し替えれば他人のリソースを操作できてしまう**（IDOR）。
- 認可は「このユーザーが、この対象に、この操作を許されているか」を1か所に集約する。**認可は業務ルールそのもの**なので、Gate/Policy として明示しテストできる形に置くのが堅い。
- どの仕組みを使うかは用途で決まる：画面付きアプリ＝Breeze/Jetstream、SPA や同一ドメインのフロント＝**Sanctum（クッキーセッション）**、外部向けAPIトークン＝Sanctum のトークン、本格OAuthサーバ＝Passport。

## 基本の書き方（コード）
### 認証：スターターキット / Sanctum
```bash
# 画面付きの最小構成（ログイン/登録/パスワードリセット一式）
composer require laravel/breeze --dev
php artisan breeze:install        # blade / react / vue / api から選ぶ

# SPA・APIトークン用
composer require laravel/sanctum
php artisan install:api           # Laravel 11 はこれで sanctum 周りを整える
```
```php
// 手動ログイン（自前コントローラ）：Auth ファサード
use Illuminate\Support\Facades\Auth;

if (Auth::attempt(['email' => $email, 'password' => $password])) {
    $request->session()->regenerate();   // セッション固定攻撃対策で必ず再生成
    return redirect()->intended('/dashboard');
}
// 現在ユーザー / ログアウト
$user = Auth::user();   // または $request->user()
Auth::logout();

// APIトークンの発行（Sanctum）
$token = $user->createToken('mobile')->plainTextToken;  // クライアントへ返す
```

### 認可：Gate（軽量・クロージャ）
```php
// app/Providers/AppServiceProvider.php の boot() など
use Illuminate\Support\Facades\Gate;

Gate::define('update-post', function (User $user, Post $post) {
    return $user->id === $post->user_id;   // 自分の投稿だけ true
});

// 使う側
if (Gate::allows('update-post', $post)) { /* 許可 */ }
Gate::authorize('update-post', $post);     // NG なら 403 を自動送出
```

### 認可：Policy（モデル単位で集約・推奨）
```bash
php artisan make:policy PostPolicy --model=Post
```
```php
// app/Policies/PostPolicy.php
class PostPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;   // 所有者チェック
    }
    public function delete(User $user, Post $post): bool
    {
        return $this->update($user, $post);
    }
}

// コントローラ
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);   // PostPolicy@update を呼ぶ。NGなら 403
    $post->update($request->validated());
}
```
```blade
{{-- Blade で表示制御（あくまで補助。サーバ側チェックが本体） --}}
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">編集</a>
@endcan
```
Laravel 11 では Policy はモデル名から命名規約で自動解決される（`Post` → `PostPolicy`）。明示登録も `Gate::policy(Post::class, PostPolicy::class)` で可能。

## 実務での使い方・定番パターン
- **ガード/プロバイダは `config/auth.php`**：`guards`（web=セッション, sanctum=トークン）と `providers`（どのモデル・テーブルでユーザーを引くか）を定義。複数ユーザー種別（admin/member）は guard を分ける。
- **スコープで他人のレコードを引かせない**（認可の第一歩）：`$request->user()->posts()->findOrFail($id)` のように所有者起点で絞れば、Policy 以前に URL 改ざんが届かない。
- **`authorize()` を徹底**＋`before` フックで管理者を一括許可：`PostPolicy` に `before(User $user)` を置き `$user->is_admin ? true : null` で全権を抜けさせる。
- **ルートに `->middleware('can:update,post')`** を付ければコントローラ前で認可できる。一覧系は明示的に絞る（`@can` だけに頼らない）。
- **APIは `auth:sanctum` ミドルウェア**でガード。SPAは `EnsureFrontendRequestsAreStateful` でクッキーセッション認証にする。

## ハマりどころ / アンチパターン
- **認証だけして認可を忘れる（最頻・最重大）**：ログインさえすれば `Post::findOrFail($id)->update(...)` で他人の投稿を編集・削除できる（IDOR）。必ず Policy か所有者スコープで絞る。
- **ビューで非表示にしただけ＝認可ではない**：`@can` でボタンを隠してもエンドポイントは叩ける。サーバ側（`authorize()`/middleware）で必ず弾く。
- **guard の混同**：API ルートで web ガードのまま `Auth::user()` を見て常に null、あるいは `auth` と `auth:sanctum` を取り違えて 401/未認証扱い。`config/auth.php` の guard と middleware を一致させる。
- **`Auth::attempt` 後に `session()->regenerate()` を忘れる**：セッション固定攻撃のリスクが残る。ログイン直後に再生成する。
- **Policy 未登録だと黙って通る/落ちる**：命名規約から外れたクラス名は解決されない。`Gate::policy()` で明示するか規約に合わせる。
- **`Gate::allows` の戻りを無視**：`if` で確認せず処理を続けると認可が形骸化。判定結果は必ず分岐に使うか `authorize()` を使う。

## 関連
[middleware.md](./middleware.md) / [controller.md](./controller.md) / [security.md](./security.md)
