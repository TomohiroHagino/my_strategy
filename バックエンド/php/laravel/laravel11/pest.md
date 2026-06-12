# Pest（Laravel 11）

## ひとことで言うと
PHPUnit の上に乗る**関数スタイルのテストフレームワーク**。`it()` / `test()` でテストを書き、`expect()->toBe()` でアサートする。クラスや `public function test_...` を書かずに済み、Laravel 公式の `pest-plugin-laravel` で `get()` / `actingAs()` などが関数として使える。実行は `php artisan test`（中身は PHPUnit）。

## 役割・なぜ必要か
- PHPUnit の「クラス＋メソッド＋`$this->assert...`」の定型を削り、**1テスト＝1関数**で読み書きを軽くする。テストの本数が増えるほど効いてくる。
- 中身は PHPUnit なので、既存の PHPUnit テストと**共存できる**し、カバレッジ・並列実行など PHPUnit の資産をそのまま使える。
- Laravel プラグインで HTTP テストヘルパ（`get` / `post` / `actingAs` / `assertDatabaseHas`）が関数で呼べ、Feature テストが短くなる。

## 基本の書き方（コード）
```bash
# 導入（Laravel 11 の新規プロジェクトは既定で Pest 選択可）
composer require pestphp/pest pestphp/pest-plugin-laravel --dev
php artisan pest:install     # tests/Pest.php を生成
```
```php
// tests/Pest.php（全テスト共通の前提をここで宣言）
uses(
    Tests\TestCase::class,                              // Laravel の TestCase を使う
    Illuminate\Foundation\Testing\RefreshDatabase::class // 各テストでDBロールバック
)->in('Feature');   // Feature ディレクトリ配下に適用
```
```php
// tests/Unit/PriceTest.php（純ロジック：DBなし）
use App\Support\Price;

it('税込みを計算する', function () {
    expect(Price::withTax(1000, 0.10))->toBe(1100);
});

test('0円は0円のまま', function () {           // it() と test() は同義
    expect(Price::withTax(0, 0.10))->toBe(0);
});
```
```php
// tests/Feature/PostTest.php（HTTP結合：Laravelプラグイン関数を使う）
use App\Models\Post;
use App\Models\User;
use function Pest\Laravel\{get, post, actingAs};

it('一覧が200を返す', function () {
    Post::factory()->count(3)->create();
    get('/posts')->assertOk()->assertSee('投稿一覧');
});

it('ログイン済みなら作成できる', function () {
    actingAs(User::factory()->create());
    post('/posts', ['title' => 'X', 'body' => '本文'])
        ->assertRedirect('/posts');
    expect(Post::where('title', 'X')->exists())->toBeTrue();
});
```
```php
// expectation API（チェーンで条件を重ねられる）
expect([1, 2, 3])
    ->toHaveCount(3)
    ->toContain(2)
    ->not->toContain(9);
expect($user->email)->toBeString()->toContain('@');
```

## 実務での使い方・定番パターン
- **`it()` / `test()` の使い分け**：意味は同じ。`it('does X', ...)` は「it does X」と読める英語寄り、`test('説明', ...)` は日本語説明と相性が良い。チーム内で統一する。
- **datasets（データ駆動）**：同じ検証を複数入力で回す。テーブル駆動の Pest 版。
  ```php
  it('メールを検証する', function (string $email, bool $valid) {
      expect(isValidEmail($email))->toBe($valid);
  })->with([
      ['a@example.com', true],
      ['bad', false],
      'スペース' => ['  ', false],   // キー付きで失敗箇所を可視化
  ]);
  ```
- **`beforeEach` / `afterEach`**：各テスト前後の共通処理を関数で。`beforeEach(fn () => $this->user = User::factory()->create());` のように `$this` 経由でテストへ値を渡せる。
- **`tests/Pest.php` に共通設定を集約**：`uses(TestCase::class, RefreshDatabase::class)->in('Feature')` で Feature 配下に一括適用。ディレクトリごとに前提を変えられる。
- **Laravel プラグイン関数**：`get/post/put/delete`、`actingAs`、`assertDatabaseHas` などが `use function Pest\Laravel\...` で関数として使える。`$this->get()` でも書けるが、関数形のほうが短い。
- **実行は PHPUnit と同じ**：`php artisan test` / `--filter` / `--parallel` / `--coverage --min=80` がそのまま効く。`./vendor/bin/pest` でも実行可。
- **PHPUnit との共存**：既存の `extends TestCase` なクラス型テストと同居できる。段階移行が可能で、全部書き直す必要はない。

## ハマりどころ
- **`uses()` の付け忘れ**：`tests/Pest.php` で `RefreshDatabase` を `->in('Feature')` し忘れると、DBを使う Feature テストで前のデータが残り順序依存で落ちる。逆に Unit に DB トレイトを噛ませると無駄に遅くなる。
- **`$this` の文脈**：Pest のクロージャ内 `$this` は背後の TestCase インスタンス。`beforeEach` で `$this->x = ...` した値は同テスト内で参照できるが、クロージャを通常関数に切り出すと `$this` が外れて使えない。
- **datasets の遅延評価**：`->with([...])` の値は各ケースで展開される。`User::factory()->create()` を dataset 内に直書きすると、データ収集フェーズでDBに触れて落ちることがある。ファクトリは**クロージャで遅延**させる（`fn () => User::factory()->create()`）。
- **`expect()` と PHPUnit assert の混在**：どちらも使えるが、失敗メッセージの形式が変わる。チームで `expect` 系に寄せると読みやすい。
- **`it()` の説明文重複**：同名の `it('...')` が同ファイルにあると区別しづらい。説明は具体的に書く。
- **過剰なモック**：境界（メール・課金・外部HTTP）だけ `Mail::fake()` / `Http::fake()` で止め、自前ロジックは実物を通す。内部実装まで縛ると「もろい」テストになる。

## 関連
[testing.md](./testing.md) / [database.md](./database.md) / [artisan.md](./artisan.md)
