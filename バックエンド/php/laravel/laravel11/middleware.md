# ミドルウェア（Middleware）（Laravel 11）

## ひとことで言うと
HTTPリクエストがコントローラに届く**前後**に挟む、共通の通過処理（フィルター）。認証・CSRF・throttle（流量制限）などを「層」として重ねる仕組み。

## 役割・なぜ必要か
- 「ログインしていなければ弾く」「CSRFトークンを検証する」「1分間に60回まで」など、**多くのルートで共通して必要な前処理／後処理**を、コントローラに書かずに切り出すためにある。
- リクエストは複数のミドルウェアを**たまねぎ状**に通過する（外側→内側→コントローラ→内側→外側）。各層は「次へ渡す（`$next($request)`）」か「途中で打ち切る（リダイレクト／例外）」かを選べる。
- Railsの `before_action` / Rack ミドルウェアに相当。Laravelでは「グローバル（全リクエスト）」「グループ（web/api）」「ルート個別」の3段階で適用範囲を制御する。

## 基本の書き方（コード）
```bash
php artisan make:middleware EnsureTokenIsValid
```

```php
// app/Http/Middleware/EnsureTokenIsValid.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    public function handle(Request $request, Closure $next): Response
    {
        // --- ここがコントローラ「前」の処理 ---
        if ($request->input('token') !== 'secret') {
            return redirect('/home');          // 打ち切り（次へ渡さない）
        }

        $response = $next($request);            // 次の層 → コントローラへ

        // --- ここがコントローラ「後」の処理 ---
        $response->headers->set('X-Checked', '1');
        return $response;
    }
}
```

**Laravel 11では登録が `bootstrap/app.php` に集約**された（旧 `app/Http/Kernel.php` は廃止）。
```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(/* ... */)
    ->withMiddleware(function (Middleware $middleware) {
        // 全リクエストに適用（グローバル）
        $middleware->append(\App\Http\Middleware\EnsureTokenIsValid::class);

        // エイリアス（ルートで短い名前で呼ぶ）
        $middleware->alias([
            'token' => \App\Http\Middleware\EnsureTokenIsValid::class,
        ]);

        // web / api グループへ追加
        $middleware->web(append: [\App\Http\Middleware\TrackLastSeen::class]);
    })
    ->create();
```

## 実務での使い方・定番パターン
- **ルート個別 / グループ適用**は `routes/web.php` で `->middleware()`。エイリアスや組み込み名で指定する。
  ```php
  Route::get('/dashboard', [DashboardController::class, 'index'])
      ->middleware(['auth', 'verified']);          // 認証＋メール確認済み

  Route::middleware('throttle:60,1')->group(function () {   // 1分60回まで
      Route::post('/comments', [CommentController::class, 'store']);
  });

  Route::get('/admin', AdminController::class)->middleware('token');  // 自作エイリアス
  ```
- **組み込みの定番**：`auth`（未ログインを弾く）／`auth:sanctum`（API）／`throttle:名前`（レートリミット、定義は `bootstrap/app.php` か Provider）／`verified`（メール確認）／`signed`（署名付きURL）。
- **パラメータ付き**：`role:admin` のように `handle(Request $request, Closure $next, string $role)` で第3引数以降に受け取れる。`,` 区切りで複数渡せる。
- **CSRFは web グループに標準で入っている**（`ValidateCsrfToken`）。除外したいパスは `$middleware->validateCsrfTokens(except: ['stripe/*'])` のように `bootstrap/app.php` で指定する（外部Webhook等）。
- **実行順を意識**：`auth` の後に「自分が持つ権限を見る」ミドルウェアを置く、のように依存順で並べる。グローバルの優先順位は `->withMiddleware()` 内の `priority()` で制御できる。

## ハマりどころ / アンチパターン
- **Laravel 11で登録場所が変わった**：ネット記事の多くは `app/Http/Kernel.php` の `$middlewareAliases` / `$middlewareGroups` を編集せよと書くが、**11以降そのファイルは無い**。登録はすべて `bootstrap/app.php` の `->withMiddleware()`。ここを間違えると「ミドルウェアが効かない／クラスが見つからない」になる。
- **`return $next($request)` を返し忘れる**：`handle()` が何も返さない／`$next` を呼ばないと、リクエストが先に進まず空レスポンスや500になる。後処理を入れるなら必ず `$response = $next($request)` で受けてから返す。
- **実行順の誤解**：グローバル → グループ（web/api）→ ルート個別、の順で外側から重なる。「認証より先に権限チェックが走って常に弾かれる」等は順序ミスが原因。
- **重い処理を全グローバルに入れる**：全リクエストで毎回DBアクセス…は遅い。必要なルートにだけ付ける。
- **エイリアス未登録で `middleware('token')`**：`alias()` に登録していない名前を使うと `Target class [token] does not exist` 系で落ちる。

## 関連
[controller.md](./controller.md) / [auth.md](./auth.md) / [config_env.md](./config_env.md)
