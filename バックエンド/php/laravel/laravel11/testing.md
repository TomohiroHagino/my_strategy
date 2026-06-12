# テスト（Pest / PHPUnit）（Laravel 11）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。Laravel は **PHPUnit** を従来から同梱し、新規プロジェクトでは記述の軽い **Pest**（PHPUnit の上に乗る関数スタイルのテストフレームワーク）も選べる。どちらも実行は `php artisan test`。レイヤーごとに Unit test と Feature test を書き分ける。

## 役割・なぜ必要か
- 変更のたびに手で全画面を確認するのは非現実的。**回帰（デグレ）を自動で検出**するために要る。
- 「この仕様で動く」という実行可能な仕様書になり、リファクタの安全網になる。
- Laravel はテスト用ヘルパ（HTTPテスト・DB初期化・ファクトリ）が手厚く、結合テストを書きやすい。

## 基本の書き方（コード）
```php
// tests/Feature/PostTest.php（PHPUnit スタイル：HTTP結合テスト）
namespace Tests\Feature;

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;
use App\Models\Post;

class PostTest extends TestCase
{
    use RefreshDatabase;   // 各テストでDBをロールバックし汚さない

    public function test_一覧が200を返す(): void
    {
        Post::factory()->count(3)->create();
        $response = $this->get('/posts');
        $response->assertOk()->assertSee('投稿一覧');
    }
}
```
```php
// tests/Feature/PostTest.php（Pest スタイル：同じ内容を関数で）
use App\Models\Post;
use function Pest\Laravel\get;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

it('一覧が200を返す', function () {
    Post::factory()->count(3)->create();
    get('/posts')->assertOk()->assertSee('投稿一覧');
});
```
```php
// tests/Unit/PriceTest.php（Unit：DBやフレームワークに依存しない純ロジック）
use App\Support\Price;

it('税込みを正しく計算する', function () {
    expect(Price::withTax(1000, 0.10))->toBe(1100);
});
```

## 実務での使い方・定番パターン
- **実行コマンド**：
  ```bash
  php artisan test                       # 全テスト
  php artisan test --filter=PostTest     # 名前で絞り込み
  php artisan test --parallel            # 並列実行（高速化）
  php artisan test --coverage --min=80   # カバレッジ計測＆下限ガード
  ```
- **Feature と Unit の使い分け**：
  - **Feature test**（`tests/Feature/`）= ルーティング＋ミドルウェア＋コントローラ＋DBを**通した結合テスト**。実務の主役。`$this->get()/post()` で擬似HTTPリクエストを投げ、`assertOk()` / `assertRedirect()` / `assertJson()` で検証。
  - **Unit test**（`tests/Unit/`）= 単一クラス・関数の純ロジック。DBやHTTPを使わない軽いもの。
- **`RefreshDatabase` トレイト**：各テスト前にマイグレーションを当て、各テストをトランザクションで囲んで終了時にロールバック。テスト間でDBが汚れない。SQLite メモリDB（`:memory:`）と組み合わせると速い。
- **モデルファクトリ**：テストデータは `Post::factory()->create()` で宣言的に生成。`make()`（保存しない・軽い）と `create()`（DB保存）を使い分ける。状態は `Post::factory()->published()->create()` のように state メソッドで表現。
- **HTTPテストの定番アサーション**：`assertOk()` / `assertStatus(201)` / `assertRedirect('/login')` / `assertJsonFragment([...])` / `assertSee()` / `assertDatabaseHas('posts', ['title' => 'X'])`。認証は `$this->actingAs($user)`。
- **`.env.testing`**：テスト専用の環境変数を置く。`phpunit.xml` で `APP_ENV=testing` と `DB_CONNECTION=sqlite` / `DB_DATABASE=:memory:` を指定するのが定番。これで本番・開発DBに触れない。

## ハマりどころ / アンチパターン
- **テスト間のDB状態リーク**：`RefreshDatabase` を付け忘れると前のテストのデータが残り、順序依存で落ちる。DBを使うテストには必ず付ける（または `DatabaseTransactions`）。
- **本番/開発DBを向く事故**：`phpunit.xml` や `.env.testing` でテスト用DBを指定しないと、`RefreshDatabase` が**開発DBを `migrate:fresh` 相当で吹き飛ばす**最悪ケースがある。テストは必ず専用DB（`:memory:` 推奨）を向ける。CIでも同様。
- **過剰なモック**：実装の内部呼び出しまでモックで縛ると、リファクタで赤くなる「もろい」テストに。**境界（外部API・メール・課金など）だけ**をフェイク（`Mail::fake()` / `Queue::fake()` / `Http::fake()`）し、自前ロジックは実物で通す。
- **キャッシュ/設定の持ち越し**：テスト中に `config:cache` 済みのキャッシュが残ると `.env.testing` が効かない。CIでは `config:clear` してからテストする。
- **カバレッジ偏重**：数字（80%目安）だけ追って**重要フローのFeature testが無い**のは本末転倒。まず主要なユーザー操作をFeatureで押さえる。
- **`create` の乱用で遅い**：保存不要なら `make()` / `make()->each()`。並列実行（`--parallel`）も併用して時間を抑える。

## 関連
[database.md](./database.md) / [artisan.md](./artisan.md) / [controller.md](./controller.md)
