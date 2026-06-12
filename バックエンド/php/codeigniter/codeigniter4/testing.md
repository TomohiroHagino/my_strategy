# テスト（Testing）（CodeIgniter 4）

## ひとことで言うと
アプリの振る舞いを**自動で検証するコード**。CI4は **PHPUnit** を土台に、専用の基底クラス **`CIUnitTestCase`** と各種トレイト（`FeatureTestTrait` / `DatabaseTestTrait`）を提供する。実行は `php spark test`（または `vendor/bin/phpunit`）。

## 役割・なぜ必要か
- 変更のたびに全画面を手で確認するのは非現実的。**回帰（デグレ）を自動検出**するために要る。
- 「この仕様で動く」という実行可能な仕様書になり、リファクタの安全網になる。
- CI4はHTTP結合（Feature）とDB検証（Database）を専用トレイトで簡単に書けるよう用意している。

## 基本の書き方（コード）
```php
// tests/unit/CalcTest.php（単体：CIUnitTestCase を継承）
use CodeIgniter\Test\CIUnitTestCase;

final class CalcTest extends CIUnitTestCase
{
    public function testAddReturnsSum(): void
    {
        // Arrange / Act
        $result = 1 + 2;
        // Assert
        $this->assertSame(3, $result);
    }
}
```
```php
// tests/feature/HomeTest.php（HTTP結合：FeatureTestTrait）
use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\FeatureTestTrait;

final class HomeTest extends CIUnitTestCase
{
    use FeatureTestTrait;

    public function testIndexReturns200(): void
    {
        $result = $this->get('/');                 // ルート経由で実行
        $result->assertStatus(200);
        $result->assertSee('Welcome');             // ボディに文字列があるか
    }

    public function testPostCreatesUser(): void
    {
        $result = $this->post('/users', ['name' => 'Taro']);
        $result->assertRedirectTo('/users');
    }
}
```
```php
// tests/database/UserModelTest.php（DB検証：DatabaseTestTrait）
use CodeIgniter\Test\CIUnitTestCase;
use CodeIgniter\Test\DatabaseTestTrait;
use App\Models\UserModel;

final class UserModelTest extends CIUnitTestCase
{
    use DatabaseTestTrait;

    protected $refresh    = true;                  // 各テスト前にマイグレーション再実行
    protected $migrate    = true;
    protected $namespace  = 'App';

    public function testInsertPersistsRow(): void
    {
        $model = new UserModel();
        $model->insert(['name' => 'Taro', 'email' => 't@example.com']);

        // テーブルに該当行が存在するか
        $this->seeInDatabase('users', ['email' => 't@example.com']);
        $this->dontSeeInDatabase('users', ['email' => 'none@example.com']);
    }
}
```
```php
// Fabricator でテストデータを量産（Faker連携）
use CodeIgniter\Test\Fabricator;

$fabricator = new Fabricator(UserModel::class);
$users = $fabricator->make(5);     // 5件のダミー（DB保存なし）
$fabricator->create(3);            // 3件を実DBに保存
```
```bash
# 実行
php spark test                      # 全テスト
php spark test --filter HomeTest    # 名前で絞り込み
vendor/bin/phpunit --coverage-html build/coverage   # カバレッジ（xdebug/pcov 必要）
```

## 実務での使い方・定番パターン
- **テストピラミッド**：土台に多数の高速な単体（`CIUnitTestCase`）、中間に **Feature**（ルーティング＋コントローラ＋ビューの結合、`$this->get()/post()`）、頂点に少数のE2E。
- **Feature を主役に**：`assertStatus` / `assertSee` / `assertRedirectTo` / `assertSessionHas` でHTTPの実態を検証でき、コントローラ単体テストより実態に近い。
- **DBは専用接続を分ける**：`.env` のテスト用に `tests` グループ（`database.tests.*`）または `CI_ENVIRONMENT=testing` を用意し、本番/開発DBと隔離。
- **`Fabricator`** でテストデータを宣言的に生成。`make()`（保存なし・軽い）と `create()`（保存あり）を使い分ける。→ [models.md](./models.md)
- **`refresh = true`** で各テスト前にDBをマイグレーションし直し、テスト間の状態リークを防ぐ。
- モック：HTTPやサービスは `Services::injectMock()` / `\Config\Services` の差し替えで境界だけ差し替える。

## ハマりどころ / アンチパターン
- **テストDBの設定漏れ**：`tests` DBグループや `CI_ENVIRONMENT=testing` を用意せず、開発DBに直接書き込み→データ汚染。隔離が必須。
- **トレイトの付け忘れ**：`$this->get()` には `FeatureTestTrait`、`seeInDatabase` には `DatabaseTestTrait` が必要。付けないと「メソッドが無い」エラー。
- **`.env` のテスト設定不備**：`encryption.key` 未設定や `baseURL` 未設定でFeatureテストが落ちる。テスト用の値を `.env` または `phpunit.xml` の `<env>` で渡す。
- **`refresh` 無しで状態リーク**：前テストが残した行で後続テストが順序依存に落ちる。`refresh = true` で初期化。
- **過剰なモック**：内部実装を縛りすぎるとリファクタで赤くなる「もろい」テストに。境界（外部API等）だけモックする。
- **カバレッジ偏重**：数字（80%目安）だけ追って重要フローの Feature テストが無いのは本末転倒。`xdebug`/`pcov` 未導入だとカバレッジが出ない点にも注意。

> **ライブラリ個別ファイルは作らない**：CI4 は **PHPUnit ＋ 組込の `CIUnitTestCase`**（＋ `FeatureTestTrait` / `DatabaseTestTrait` / `Fabricator`）で完結するため、追加ライブラリの解説ファイルは不要。上記の組込クラス・トレイトで Unit / Feature / Database / データ生成のすべてをまかなえる。

## 関連
[models.md](./models.md) / [database_query_builder.md](./database_query_builder.md) / [config_env.md](./config_env.md) / [routing.md](./routing.md)
