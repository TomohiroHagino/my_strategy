# コントローラ（Controller）（Laravel 11）

## ひとことで言うと
ルートから渡されたリクエストを受け取り、**入力の取り回し → 業務処理の呼び出し → レスポンス返却**をまとめる「司令塔」クラス。`app/Http/Controllers/` に置く。

## 役割・なぜ必要か
- MVCの「Controller」。ルート（対応表）と、モデル/サービス（業務ロジック）の**橋渡し**を担う。
- リクエストごとの分岐・整形をここに集約し、ルートファイルを薄く保つ。
- Laravelの方針は **Skinny Controller**：コントローラは「受けて・呼んで・返す」だけ。重い業務ルールはモデル/サービスへ。バリデーションはフォームリクエストへ追い出す。→ [validation.md](./validation.md)

## 基本の書き方（コード）
```bash
# 生成（--resource で7アクション雛形、--invokable で単一アクション）
php artisan make:controller PostController --resource
php artisan make:controller PublishPostController --invokable
```

```php
// app/Http/Controllers/PostController.php
namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;   // フォームリクエスト
use App\Models\Post;
use App\Services\PostService;             // 業務ロジックを持つサービス

class PostController extends Controller
{
    // コンストラクタインジェクション: コンテナがPostServiceを自動解決して注入
    public function __construct(private PostService $posts) {}

    // 一覧 → ビューを返す
    public function index()
    {
        $posts = Post::latest()->paginate(20);     // N+1注意。必要なら ->with('user')
        return view('posts.index', compact('posts'));
    }

    // 詳細 → ルートモデルバインディングで Post が注入される
    public function show(Post $post)
    {
        return view('posts.show', ['post' => $post]);
    }

    // 作成 → フォームリクエストで検証済みデータだけ受け取る
    public function store(StorePostRequest $request)
    {
        $post = $this->posts->create($request->validated());  // 業務はサービスへ委譲
        return redirect()->route('posts.show', $post)         // PRGパターン
                         ->with('status', '投稿を作成しました');
    }
}
```

```php
// 単一アクションコントローラ（__invoke のみ）。ルートは Route::post('/publish', PublishPostController::class);
namespace App\Http\Controllers;

use App\Models\Post;

class PublishPostController extends Controller
{
    public function __invoke(Post $post)
    {
        $post->update(['published_at' => now()]);
        return redirect()->back();
    }
}
```

```php
// 返し方いろいろ（用途で使い分け）
return view('posts.show', compact('post'));     // HTML（Blade）
return response()->json(['data' => $post]);     // JSON（API）
return redirect()->route('posts.index');        // リダイレクト
return response()->json(['message' => 'not found'], 404); // ステータス指定
```

## 実務での使い方・定番パターン
- **リソースコントローラ**: `--resource` + `Route::resource('posts', PostController::class)` で index/create/store/show/edit/update/destroy の7アクションを規約どおりに対応づけ。CRUDの定番形。→ [routing.md](./routing.md)
- **依存注入（DI）**: 型ヒントするだけでサービスコンテナが解決して注入する。`new` で手動生成しない＝差し替え・テストが楽。コンストラクタ注入（全メソッド共通）とメソッド注入（そのアクションだけ）を使い分け。→ [service_container.md](./service_container.md)
- **バリデーションはフォームリクエストへ**: `make:request` で作り、引数型ヒントするだけで自動検証＋失敗時リダイレクト。コントローラ本体から検証コードが消える。→ [validation.md](./validation.md)
- **業務はモデル/サービスへ委譲**: 複数モデルにまたがる手続きはサービスクラスに切り出し、コントローラからは1〜2行呼ぶだけにする。
- **PRG（Post-Redirect-Get）**: 書き込み後は `redirect()` で戻す。リロード二重送信を防ぐ定番。

## ハマりどころ / アンチパターン
- **Fat Controller**: 検証・SQL・メール送信・整形を全部コントローラに詰める→テスト不能・再利用不可。検証→フォームリクエスト、業務→サービス/モデル、整形→Blade/アクセサに分散させる。
- **二重レスポンス**: `return` し忘れて `view()` を呼ぶだけ、あるいは `redirect()` 後に処理を続けて別レスポンスを生成しようとする→意図しない出力。アクションは**必ず1つのレスポンスを return** する。
- **ルートモデルバインディングを使わず手動 `find`**: `Post::find($id)` を毎回書くのは冗長。型ヒント `Post $post` に任せる（404も自動）。→ [routing.md](./routing.md)
- **`request()->all()` を直接DBへ**: mass assignment 脆弱性。`$request->validated()` か、モデルの `$fillable` を必ず併用。→ [security.md](./security.md)
- **N+1**: 一覧で `$post->user->name` を回す前に `->with('user')` で eager load。コントローラの取得方法で性能が決まる。→ [database.md](./database.md)
- **コントローラにビューの都合（HTML整形）を書く**: 表示整形はBladeやアクセサ/ビューモデルへ。コントローラは制御に専念。

## DTO（データ運搬オブジェクト）はどこ？

Laravel も Eloquent モデルをそのまま使い回すのが基本だが、DTOの役割を果たす仕組みが**標準＋ジェネレータ付き**で揃っている（Spring のように自然と分離できる）。

| DTOの役割 | Laravel での担当 |
|---|---|
| 出力（API/JSON） | **API Resource**（`make:resource` → `JsonResource` の `toArray()`）＝レスポンスDTOの定番 |
| 入力（検証・整形） | **Form Request**（`make:request` → `rules()` / `$request->validated()`）＝リクエストDTO相当 |
| 明示的な型付きDTO | **spatie/laravel-data**（定番gem）／ 素の `readonly` クラス（PHP 8.1+） |
| 画面表示の整形 | View Composer / Blade（＋必要なら Presenter） |

```php
// 出力：Eloquentを直接返さず Resource を挟む（出力の形を制御）
return UserResource::collection(User::paginate());

// app/Http/Resources/UserResource.php
public function toArray($request): array {
    return ['id' => $this->id, 'name' => $this->name];  // 必要な項目だけ返す
}
```
```php
// 入力：Form Request で検証済みデータだけ受け取る（＝リクエストDTO）
public function store(StorePostRequest $request) {
    $post = Post::create($request->validated());
}
```
- **API を返すときは Eloquent を直接返さず、必ず `Resource` を挟む**のが定番。入力は **`FormRequest`** で検証してコントローラを薄く保つ。→ [validation.md](./validation.md)
- Spring / Laravel は「DTOの仕組みが標準で前のめり」、Rails / Hanami は「兼ねて必要時に足す」寄り。

## 関連
[routing.md](./routing.md) / [model.md](./model.md) / [validation.md](./validation.md) / [service_container.md](./service_container.md)
