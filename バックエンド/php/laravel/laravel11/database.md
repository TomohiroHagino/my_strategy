# DB（マイグレーション / クエリ / リレーション / N+1）（Laravel 11）

## ひとことで言うと
LaravelのDB層は **マイグレーション（スキーマをPHPで記述・バージョン管理）**、**クエリビルダ（DB::table）**、**Eloquent（ORM）** の3本柱。SQLをほぼ書かずにPHPでDBを操作する。

## 役割・なぜ必要か
- DBアクセスを「PHPのメソッド呼び出し」に翻訳し、生SQLの重複・ミス・**SQLインジェクション**を避けるためにある（バインドパラメータが自動で効く）。
- スキーマ管理・クエリ・リレーション・トランザクション・テストデータ生成（factory/seeder）までを一手に担う。

---

## マイグレーションとは
スキーマ変更を**PHPで記述しバージョン管理**する仕組み。`.env` 既定はSQLite（Laravel 11）。
```php
// database/migrations/xxxx_create_posts_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('body')->nullable();
            $table->boolean('published')->default(false);
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->timestamps();           // created_at / updated_at
            $table->index('title');
        });
    }
    public function down(): void { Schema::dropIfExists('posts'); }
};
```
```bash
php artisan make:migration add_x_to_posts --table=posts
php artisan migrate            # 未実行分を適用
php artisan migrate:rollback   # 直近バッチを戻す（down 実行）
php artisan migrate:fresh      # 全テーブルdrop → 再構築（※データ全消し）
php artisan migrate:fresh --seed   # 再構築 + シーダ投入（開発用）
```

## クエリビルダ / Eloquent クエリ
```php
// クエリビルダ（モデル不要・素の行）
DB::table('posts')->where('published', true)->orderByDesc('created_at')->get();

// Eloquent（モデル経由）
Post::where('published', true)->latest()->limit(10)->get();
Post::find(1);                    // 無ければ null
Post::findOrFail(1);              // 無ければ 404 (ModelNotFound)
Post::firstWhere('slug', 'x');
Post::published()->pluck('title');   // ローカルスコープ + 1列だけ
```

## リレーションとは
テーブル間の関係を宣言し、`$user->posts` のように辿れるようにする。
```php
class User extends Model {
    public function posts() { return $this->hasMany(Post::class); }
}
class Post extends Model {
    public function user() { return $this->belongsTo(User::class); }
    public function tags() { return $this->belongsToMany(Tag::class); } // 多対多
}
```

## N+1問題とは（最頻の性能バグ）
一覧表示で関連を1件ずつ引き、SQLが `1 + N` 回走る性能劣化。
```php
// NG: foreach ($posts as $p) { echo $p->user->name; } で N+1
$posts = Post::all();

// OK: with() で eager load（事前に2クエリでまとめ読み）
$posts = Post::with('user')->get();

// 取得済みコレクションに後から積むなら load() / loadMissing()
$posts->load('user', 'tags');
$posts->loadMissing('user');   // 未ロードのものだけ（二重ロード回避）
```
**対策の選択肢（使い分け）**:
- **`with('user')`** … クエリ時に eager load（最も一般的）。
- **`load('user')` / `loadMissing('user')`** … 取得済みモデルに後付けで eager load。
- **ネスト** `with('posts.comments')`、**制約付き** `with(['posts' => fn ($q) => $q->where('published', true)])`。
- **件数だけ** `withCount('posts')` … 関連を全ロードせず `posts_count` を1クエリで付与（`$post->posts_count`）。
```php
$users = User::withCount('posts')->with('profile')->get();
```
- **検出**：**Laravel Debugbar** / Telescope でクエリ数。`Model::preventLazyLoading()`（`AppServiceProvider` の `boot` で、開発中は遅延ロードを例外化して炙り出す）。

## トランザクションとは
```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    $order->save();
    $stock->decrement('count');   // どれか失敗で全部ロールバック
});
// 手動: DB::beginTransaction() / commit() / rollBack()
```

## シーダ / ファクトリ
```php
// database/factories/PostFactory.php → fake データ定義
Post::factory()->count(50)->create();            // テスト/開発の量産
php artisan db:seed                              // DatabaseSeeder 実行
```

## ハマりどころ / アンチパターン
- **N+1**（最頻）→ `with()` / `load()`。
- **`migrate:fresh` / `migrate:refresh` はデータ全消し**。本番で絶対に叩かない（事故多発）。本番は前進専用マイグレーションのみ。
- **本番マイグレーションの実行順・ロック時間**: 大テーブルへの `ALTER` はロックで停止リスク。段階移行（まず追加→移行→後で削除）。
- **`find` と `findOrFail` の取り違え**: 前者は無いと `null`（NPE誘発）、後者は 404。
- **クエリビルダの生 `whereRaw` / 文字列連結**で SQLi。バインド（`?` / `:name`）を使う。→ [security.md](./security.md)
- **`insert()` / `upsert()` はモデルイベント・キャストを通さない**（一括投入は速いが副作用が走らない）。
- **タイムゾーン**: `now()` は `config/app.php` の timezone 依存。保存はUTC・表示で変換が定石。

## Eloquent 特有の現象と対策（N+1以外）
| 現象 / 罠 | なぜ起きる | 対策 |
|---|---|---|
| 大量データでメモリ枯渇 | `Post::all()` / `->get()` は全件をメモリに展開 | **`chunk(200, fn)` / `chunkById` / `cursor()`（ジェネレータ・1件ずつ）/ `lazy()`**。`each` で逐次処理 |
| マスアサインメント脆弱性 | `Post::create($request->all())` で許可外カラムまで書ける | モデルに **`$fillable`**（許可リスト）/ `$guarded`。`$request->validated()` を渡す |
| コレクションでPHP側フィルタ | `Post::all()->where('active', true)` は**全件ロード後**にPHPで絞る | **クエリビルダ側** `Post::where('active', true)->get()` でSQLに絞らせる |
| `whereHas` が重い | 関連存在チェックがサブクエリ（相関）になり大テーブルで遅い | 必要なら `join` に書き換え、存在だけなら `whereHas` でも `withExists`/件数最適化を検討 |
| アクセサで N+1 | アクセサ（`getXAttribute`）内で関連を参照すると一覧でN+1 | アクセサが触る関連を **eager load 前提**にする（`with`） |
| 不要な全カラム取得 | `->get()` は全列 | **`select(['id','title'])`** で必要列だけ |
| `firstOrCreate`/`updateOrCreate` の競合 | 同時実行で重複INSERT・ユニーク制約違反 | DBにユニーク制約＋例外捕捉、必要ならトランザクション/リトライ |
| デッドロック | 複数行更新の順序競合 | `DB::transaction($closure, $attempts)` の第2引数で**自動リトライ回数**を指定 |
| `find` と `findOrFail` の取り違え | `find` は無いと `null`（NPE誘発）、`findOrFail` は 404 | 取得保証が要る所は `findOrFail` |
| `migrate:fresh`/`refresh` でデータ全消し | 本番で叩くと全削除 | 本番は**前進専用マイグレーションのみ**。大テーブル `ALTER` は段階移行 |

## 関連
[model.md](./model.md)
