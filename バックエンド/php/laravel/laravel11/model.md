# モデル（Eloquent Model）（Laravel 11）

## ひとことで言うと
DBのテーブル1つに対応するクラスで、**データの入れ物＋そのデータに関するビジネスロジックの置き場**。`Illuminate\Database\Eloquent\Model` を継承する **Active Record パターン**のORM。

## 役割・なぜ必要か
- テーブル `posts` ↔ クラス `Post`、1行 ↔ 1インスタンス、という対応で「SQLを直接書かずにPHPでDBを扱う」ためにある。
- MVCの中で「**何を・どう扱うか（業務ルール）**」を担う中心。コントローラやビューに業務ルールを散らさず、ここへ集約することで再利用・テストがしやすくなる。
- 1モデル = 1インスタンスが自分の保存・更新・削除を知っている（`$post->save()` / `$post->delete()`）のが Active Record の特徴。Rails の ActiveRecord と思想は同じ。

## 基本の書き方（コード）
```bash
# モデル + マイグレーションを同時生成
php artisan make:model Post -m
# さらに factory / seeder / controller もまとめて欲しいとき
php artisan make:model Post -mfsc
```
```php
// app/Models/Post.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Builder;

class Post extends Model
{
    // mass assignment 許可リスト（これ以外は create()/fill() で代入されない）
    protected $fillable = ['title', 'body', 'published'];

    // 型キャスト（DBの文字列 ↔ PHPの型を自動変換）
    protected $casts = [
        'published'    => 'boolean',
        'published_at' => 'datetime',
        'meta'         => 'array',   // JSON カラム ↔ PHP配列
    ];

    // リレーション（belongsTo / hasMany など）
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    // アクセサ（読み取り時に整形）: $post->title が title_upper 風に加工される例
    protected function title(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => trim($value),  // ミューテタ（保存時に整形）
        );
    }

    // ローカルスコープ（よく使うクエリに名前をつける）→ Post::published() で呼べる
    public function scopePublished(Builder $query): void
    {
        $query->where('published', true);
    }
}
```

## 実務での使い方・定番パターン
- **$fillable / $guarded で mass assignment を制御**。`Post::create($request->all())` のとき、`$fillable` に無い列は無視される（管理者フラグ等の上書きを防ぐ）。基本は `$fillable`（ホワイトリスト）を明示。
- **$casts** で `boolean` / `datetime` / `array`（JSON）/ enum を自動変換。Laravel 11 は enum キャストも素直に効く。
- **アクセサ/ミューテタは `Attribute::make(get:, set:)`**（旧 `getXxxAttribute` は非推奨方向）。
- **リレーション**で `$user->posts` のように辿れる。`hasMany` / `belongsTo` / `belongsToMany`（中間テーブル）/ `hasOne` / `morphTo`（ポリモーフィック）。
- **ローカルスコープ**でクエリを意味のある名前に。チェーン可（`Post::published()->latest()->get()`）。
- **モデルイベント / Observer** で保存前後のフックを書く（`php artisan make:observer PostObserver --model=Post`）。重い副作用はジョブへ逃がす。
- `timestamps` は既定で有効（`created_at` / `updated_at` 自動更新）。不要なら `public $timestamps = false;`。

## ハマりどころ / アンチパターン
- **mass assignment 脆弱性**: `$guarded = []`（全許可）で `$request->all()` を流すと、`is_admin` 等を外部から書き換えられる。必ず `$fillable` を絞る。→ [security.md](./security.md)
- **N+1**: 一覧で `$posts->each(fn($p) => $p->user->name)` は SQL が 1+N 回走る。`with()` で eager load する。→ [database.md](./database.md)
- **fat model（神モデル）**: 1モデルに数百行の業務手続きを詰める。複数モデルにまたがる手続きは Action / Service クラスへ分離。
- **Observer にメール送信・外部API**を詰めて隠れた副作用にする → デバッグ困難。重い処理はキューへ。→ [queues.md](./queues.md)
- **`save()` の戻り値を見ない**: 失敗時 `false` を返すだけ（例外でない）。検証は FormRequest 側で先に弾くのが定石。→ [validation.md](./validation.md)
- **キャスト忘れ**: `published` が文字列 `"0"`（truthy）扱いになりバグる。`$casts` に `boolean` を入れる。

## 関連
[database.md](./database.md) / [controller.md](./controller.md)
