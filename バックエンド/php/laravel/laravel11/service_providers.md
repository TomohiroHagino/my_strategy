# サービスプロバイダ（Laravel 11）

## ひとことで言うと
**アプリ起動時（bootstrap）に「何をコンテナに登録するか」を書く場所**。Laravelの初期化はすべてサービスプロバイダ経由で行われる。自作サービスやパッケージをコンテナに組み込む入口。

## 役割・なぜ必要か
- サービスコンテナへの**バインド登録の置き場**。`bind` / `singleton` をどこかに書く必要があり、その正式な場所がプロバイダ。
- Laravel本体の機能（ルーティング・キャッシュ・DB等）も全部プロバイダで登録されている。アプリは `AppServiceProvider` を中心に拡張する。
- **`register()` と `boot()` の2フェーズ**に分かれているのが肝。「まず全プロバイダの register を回し、その後 boot を回す」という順序保証があるから、boot では他サービスが登録済みと前提できる。
- → 依存の登録（register）と、登録済み依存を使う初期化（boot）を分離することで起動順の事故を防ぐ。

## 基本の書き方（コード）
```php
// app/Providers/AppServiceProvider.php
namespace App\Providers;

use App\Contracts\PaymentGateway;
use App\Services\StripeGateway;
use Illuminate\Support\ServiceProvider;
use Illuminate\Pagination\Paginator;

class AppServiceProvider extends ServiceProvider
{
    // register: コンテナへのバインドだけ。他サービスに触れない。
    public function register(): void
    {
        $this->app->singleton(PaymentGateway::class, fn ($app) =>
            new StripeGateway(config('services.stripe.secret'))
        );
    }

    // boot: 全プロバイダの register 完了後に呼ばれる。初期化処理はここ。
    public function boot(): void
    {
        Paginator::useBootstrapFive();      // 他サービス利用OK
        \Illuminate\Support\Facades\Gate::define('admin', fn ($user) => $user->is_admin);
    }
}
```

```php
// Laravel 11: プロバイダの登録は bootstrap/providers.php（← ここが変更点）
// 旧来の config/app.php の 'providers' 配列は廃止された
return [
    App\Providers\AppServiceProvider::class,
    App\Providers\PaymentServiceProvider::class, // 自作を足すならここに追記
];
// artisan make:provider PaymentServiceProvider で作ると自動で追記される
```

```php
// 遅延プロバイダ: 実際に使われるまで生成しない（起動を軽くする）
use Illuminate\Contracts\Support\DeferrableProvider;

class PaymentServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(PaymentGateway::class, fn () => new StripeGateway());
    }

    // この型が要求されたとき初めてプロバイダが起動する
    public function provides(): array
    {
        return [PaymentGateway::class];
    }
}
```

## 実務での使い方・定番パターン
- **小規模なら `AppServiceProvider` に集約**でよい。増えてきたら機能ごとにプロバイダを分割（`PaymentServiceProvider` 等）し `bootstrap/providers.php` に登録。
- **`register()` にはバインドだけ**。`config()` 読み込みやバインド定義はOKだが、他サービスのメソッド呼び出しはしない。
- **`boot()` で「他が揃っている前提の初期化」**：Gate/Policy 定義、Blade ディレクティブ追加、マクロ登録、ビューコンポーザ、モデルイベント購読など。
- **パッケージ開発**ではプロバイダで `loadRoutesFrom` / `loadMigrationsFrom` / `publishes` を使い、設定やマイグレーションを利用側へ公開する。
- 重い／たまにしか使わないサービスは**遅延プロバイダ**（`DeferrableProvider` + `provides()`）で起動コストを下げる。

## ハマりどころ / アンチパターン
- **`register()` で他サービスに依存**：まだ register フェーズの途中で、依存先が未登録かもしれない。他サービス利用は必ず `boot()` へ。これが最頻出バグ。
- **boot と register の取り違え**：バインド定義を boot に書くと「register 段階で解決しようとした別プロバイダ」から見えず失敗することがある。バインド＝register、初期化＝boot を徹底。
- **Laravel 11 で登録場所を間違える**：`config/app.php` に providers を足そうとしても効かない（廃止）。**`bootstrap/providers.php`** に書く。
- **遅延プロバイダで `boot()` を使う**：遅延プロバイダは `boot()` を持てない（呼ばれる保証がない）。初期化が要るなら遅延にしない。
- **プロバイダに業務ロジックを書く**：プロバイダは「配線（登録・初期化）」専用。ドメイン処理はサービスクラスへ。

## 関連
[service_container.md](./service_container.md) / [facades.md](./facades.md)
