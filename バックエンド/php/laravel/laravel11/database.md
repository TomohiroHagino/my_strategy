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

// 取得済みコレクションに後から積むなら load()
$posts->load('user', 'tags');
```
- 検出は **Laravel Debugbar** や `Model::preventLazyLoading()`（遅延ロードを禁止して炙り出す）。

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

## 関連
[model.md](./model.md)
