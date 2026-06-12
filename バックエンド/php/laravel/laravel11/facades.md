# ファサード（Laravel 11）

## ひとことで言うと
**コンテナ内のサービスへ「静的メソッド呼び出し風」にアクセスするための窓口**。`Cache::get()` や `Route::get()` のように書けるが、実体は静的メソッドではなく、内部でコンテナからインスタンスを解決して呼んでいる。

## 役割・なぜ必要か
- Laravelの定番API（`Cache` / `Route` / `DB` / `Log` / `Storage` / `Gate` …）はほぼファサード。**短く読みやすい記法**でコンテナのサービスを使えるのが利点。
- 仕組み：ファサードは `getFacadeAccessor()` でコンテナの「キー」を返し、静的呼び出しは `__callStatic` で**そのキーに紐づく実体オブジェクトのメソッド呼び出し**に変換される。つまり `Cache::get()` ≒ `app('cache')->get()`。
- だから見た目は静的だが**裏ではDIされたインスタンス**。テストではこのインスタンスをモックに差し替えられる（ここが「ただの static」と決定的に違う）。
- Railsの `Rails.cache` のようなグローバル参照に近い使い心地を、コンテナの上で実現したもの。

## 基本の書き方（コード）
```php
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Route;

Cache::put('key', 'value', 60);
$value = Cache::get('key');

$users = DB::table('users')->where('active', true)->get();

// ルート定義もファサード（routes/web.php）
Route::get('/users', [UserController::class, 'index']);
```

```php
// 自作ファサードの中身（イメージ）。accessor がコンテナのキー。
namespace App\Facades;

use Illuminate\Support\Facades\Facade;

class Payment extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return \App\Contracts\PaymentGateway::class; // コンテナで解決されるキー
    }
}
// Payment::charge(1000) → app(PaymentGateway::class)->charge(1000) と等価
```

```php
// リアルタイムファサード: 既存クラスを Facades\ プレフィックス付きで import すると
// その場でファサードのように静的呼び出しできる（自作ファサード不要）
use Facades\App\Services\PaymentGateway;

PaymentGateway::charge(1000); // 実体はコンテナ解決された PaymentGateway インスタンス
```

```php
// テスト: ファサードはモック可能（static なのに差し替えられるのが強み）
use Illuminate\Support\Facades\Cache;

Cache::shouldReceive('get')->once()->with('key')->andReturn('mocked');
// 以後 Cache::get('key') は 'mocked' を返す（実キャッシュに触らない）

// 配列ドライバ等に切り替える系は fake ヘルパも使える
\Illuminate\Support\Facades\Queue::fake();
\Illuminate\Support\Facades\Mail::fake();
```

## 実務での使い方・定番パターン
- **フレームワーク標準ファサード**（Cache/DB/Log/Storage/Gate/Route）は素直に使ってよい。短く読みやすい。
- **テストでは `shouldReceive()` / `fake()`** で外部副作用（キュー投入・メール送信・キャッシュ）を遮断し、`assertSent` / `assertPushed` で「呼ばれたこと」を検証するのが定番。
- **自作サービスをファサード化したい**ときは「自作ファサードクラス（`getFacadeAccessor`）」か、手軽な**リアルタイムファサード**（`Facades\` プレフィックス import）を使う。
- **DIとの使い分け**：再利用・テスト・差し替えを重視するクラス本体（サービス・コントローラ）では**コンストラクタ注入を優先**。ファサードはルート定義やちょっとした記述、Bladeに近い手続き的コードで便利。

## ハマりどころ / アンチパターン
- **依存が隠れる**：クラスがどのサービスに依存しているかコンストラクタを見ても分からなくなる。多用するとテスト容易性・可読性が落ちる。重要クラスはDIにする。
- **ファサード乱用**：何でも `Facade::` で呼ぶと、実質グローバル状態への依存だらけになり結合度が上がる。境界（コントローラの薄い処理・設定的コード）に留める。
- **「static だから速い/単純」という誤解**：実体はコンテナ解決＋プロキシ。挙動はインスタンスメソッドなので、内部状態やモック可能性を意識する。
- **テストでモックし忘れ**：`Mail::send()` 等を fake せずにテストすると実送信・実書き込みが走る。外部副作用ファサードは必ず `fake()`。
- **自作クラスを安易にファサード化**：注入できる場面でファサードにすると依存が不可視化する。まずDIを検討し、本当に呼び出しを短くしたい所だけファサード/リアルタイムファサードに。

## 関連
[service_container.md](./service_container.md) / [service_providers.md](./service_providers.md)
