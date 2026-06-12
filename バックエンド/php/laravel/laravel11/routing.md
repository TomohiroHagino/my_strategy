# ルーティング（Routing）（Laravel 11）

## ひとことで言うと
URL（とHTTPメソッド）を**どの処理（コントローラ/クロージャ）に渡すか**を定義する対応表。`routes/web.php`（画面系）と `routes/api.php`（API系）に書く。

## 役割・なぜ必要か
- 「`GET /posts/1` が来たら `PostController@show` を呼ぶ」というリクエストの入口を一元管理する。
- 名前付きルート・ルートモデルバインディング・ミドルウェア適用など、**URLにまつわる定型処理をまとめて宣言的に書ける**。
- Laravel 11 では **API ルートは既定で無い**。`php artisan install:api` を実行すると `routes/api.php` と Sanctum が導入され、`bootstrap/app.php` に `api: __DIR__.'/../routes/api.php'` が登録される。

## 基本の書き方（コード）
```php
// routes/web.php
use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

// クロージャ（小さな処理向け）
Route::get('/', fn () => view('welcome'));

// コントローラのアクションへ（[クラス, メソッド] 配列で指定）
Route::get('/posts', [PostController::class, 'index']);
Route::post('/posts', [PostController::class, 'store']);

// 名前付きルート（->name() で参照名をつける）
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');

// ルートモデルバインディング: {post} を Post モデルに自動解決（見つからなければ404）
Route::get('/posts/{post}/edit', [PostController::class, 'edit']);

// グループ: prefix / middleware / name をまとめて適用
Route::middleware('auth')
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
        // 名前は admin.dashboard、URLは /admin/dashboard
    });

// リソースコントローラ: index/create/store/show/edit/update/destroy を一括定義
Route::resource('photos', PhotoController::class);
Route::apiResource('articles', ArticleController::class); // API向け（create/edit無し）
```

```php
// 名前付きルートの参照（ハードコードURLを避ける）
route('posts.show', ['post' => $post]);   // /posts/1
redirect()->route('admin.dashboard');
```

```blade
{{-- Blade内 --}}
<a href="{{ route('posts.show', $post) }}">詳細</a>
```

```bash
# 定義済みルート一覧（メソッド・URI・名前・アクションを確認）
php artisan route:list
php artisan route:list --path=posts   # 絞り込み

# 本番のルート高速化（クロージャは不可になる点に注意）
php artisan route:cache
php artisan route:clear
```

## 実務での使い方・定番パターン
- **ルートモデルバインディング**: `{post}` という名前と型ヒント `Post $post` を一致させると、Laravelが `Post::findOrFail()` 相当を自動実行。コントローラからID取得・検索の定型コードが消える。`{post:slug}` でキー列も変えられる。→ [controller.md](./controller.md)
- **名前付きルート徹底**: URL文字列を直書きせず必ず `route('...')`。URL構造を変えても参照側が壊れない。
- **グループでミドルウェア集約**: 認証必須エリアは `Route::middleware('auth')->group(...)` でまとめる。`prefix` `name` も同時付与で重複を排除。
- **`route:list` で全体把握**: 既存プロジェクト参画時、まずこれでエンドポイント一覧を眺めるのが定石。
- **APIは別ファイル**: `routes/api.php` は自動で `/api` プレフィックス＋`api` ミドルウェアグループ（ステートレス）。`web.php` はセッション・CSRF付き。役割で分ける。

## ハマりどころ / アンチパターン
- **`route:cache` 後にクロージャルートが落ちる**: `Route::get('/', fn () => ...)` のようなクロージャはキャッシュ不可で例外になる。本番でキャッシュするなら**すべてコントローラ参照に書き換える**。
- **ミドルウェアの適用順**: グループのネストや `middleware()` の指定順で実行順が変わる。認証→認可の順序を間違えると意図しない通過・拒否が起きる。`bootstrap/app.php` の `->withMiddleware()` で順序を制御。→ [middleware.md](./middleware.md)
- **同一URI・同一メソッドの重複定義**: 後勝ちで上書きされ、意図しないアクションが呼ばれる。`route:list` で重複を確認。
- **`api.php` が無いのに API ルートを書こうとする**: 11では `php artisan install:api` が前提。これを忘れて404を悩むのが定番。
- **ルートモデルバインディングの名前不一致**: `{post}` なのに引数が `$article` だと解決されず `null` が注入される。URI変数名とコントローラ引数名を一致させる。
- **`web.php` に大量のロジックを書く**: ルートはあくまで対応表。処理はコントローラ（さらに業務はサービス/モデル）へ。→ [controller.md](./controller.md)

## 関連
[controller.md](./controller.md) / [middleware.md](./middleware.md)
