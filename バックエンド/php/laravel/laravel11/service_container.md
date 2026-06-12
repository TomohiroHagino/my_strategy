# サービスコンテナ / DI（Laravel 11）

## ひとことで言うと
クラスが動くのに必要な「道具（依存オブジェクト）」を、**自分で `new` して用意する代わりに、Laravelが型ヒントを見て自動で組み立てて渡してくれる仕組み**。その道具を登録・生成する“箱”がサービスコンテナ。

> ひとことイメージ:「`ReportController` をくれ」と頼むと、コンテナが
> 「これには決済ゲートウェイが要るな → それにはHTTPクライアントとロガーが要るな…」と
> **芋づる式に全部たどって組み立て、完成品を渡してくれる**自動工場。

## 役割・なぜ必要か

### まず「無い世界」の不便を見る
コンテナが無いと、依存が依存を呼ぶので**自分で全部 `new` して積み上げる**ことになる:
```php
// ReportController を1個作るために…
$logger  = new FileLogger('/var/log/app.log');
$http    = new HttpClient($config);
$gateway = new StripeGateway($http, $logger, $apiKey); // gatewayはhttpとloggerが要る
$controller = new ReportController($gateway);          // controllerはgatewayが要る
```
- 1個使いたいだけなのに、その**下にぶら下がる道具まで全部手で組む**必要がある。
- しかも `StripeGateway` を別の決済実装に変えたくなったら、**`new` している箇所を全部探して直す**羽目になる。

### コンテナが有る世界
やることは「**コンストラクタに型を書くだけ**」。組み立てはコンテナがやる:
```php
class ReportController extends Controller
{
    // 「PaymentGateway が要る」と型で宣言するだけ。
    // Laravel が StripeGateway も、その先の HttpClient / Logger も
    // 芋づる式に作って、ここに完成品を注入してくれる。
    public function __construct(private PaymentGateway $gateway) {}

    public function show()
    {
        return $this->gateway->fetch();
    }
}
```
この「**自分で new せず、外から入れてもらう**」やり方を **DI（依存性の注入）**、それを自動でやる箱が **サービスコンテナ（IoCコンテナ）** です。

### 何が嬉しいか（3つ）
1. **組み立てから解放**される（芋づるはコンテナ任せ）。
2. **差し替えできる**：「`PaymentGateway` という“約束（インターフェース）”に対して、本番は Stripe、テストは偽物」を1行の登録で切り替えられる。
3. **テストが楽**：本物の決済を呼ばず、偽の実装を注入して検証できる。

> Rails には「コンテナ」という明示的な箱は薄い（自前 new やグローバル参照が多い）。
> **依存を中央の箱で管理する**のが Laravel の土台思想で、ここが Rails と一番違う肝。

## 基本の書き方（コード）
レベル順に3段階で覚えると迷わない。

### レベル1: 何も登録せず、型ヒントするだけ（autowiring・これで9割）
```php
// 具体クラスを型ヒントすれば、登録ゼロでLaravelが勝手に生成・注入する
public function __construct(private OrderService $orders) {}
```
コントローラ・ジョブ・イベントリスナ・コマンドなど、**Laravelが生成する場所なら基本これだけ**で動く。

### レベル2: インターフェースに実装を結びつける（差し替えの肝）
型ヒントが「具体クラス」でなく「インターフェース（＝約束）」のときは、
**どの実装を渡すかを1回だけ登録**する。書く場所はサービスプロバイダの `register()`（→ [service_providers.md](./service_providers.md)）。
```php
use App\Contracts\PaymentGateway;   // 約束（インターフェース）
use App\Services\StripeGateway;     // 実装

$this->app->bind(PaymentGateway::class, StripeGateway::class);
// → 以後 PaymentGateway を型ヒントした全箇所に StripeGateway が入る。
//   テスト時に FakeGateway へ bind し直せば、全箇所がまとめて差し替わる。
```

### レベル3: 使い分けと小技
```php
// bind = 解決のたびに新品を作る / singleton = 初回だけ作って使い回す
$this->app->bind(Foo::class, fn ($app) => new Foo());        // 毎回 new
$this->app->singleton(PaymentGateway::class, fn ($app) =>     // アプリ内で1個
    new StripeGateway(config('services.stripe.key'))
);

// 型ヒントできない場所（ルートのクロージャ内など）では手動で取り出す
$gateway = app(PaymentGateway::class);          // ヘルパ
$gateway = app()->make(PaymentGateway::class);  // 同じ意味

// 呼び出し元によって実装を変える（コンテキスト依存バインディング）
$this->app->when(PhotoController::class)
          ->needs(Filesystem::class)
          ->give(fn () => Storage::disk('s3'));  // この時だけ s3 を注入
```

## 実務での使い方・定番パターン
- **基本はコンストラクタに型ヒント**（レベル1）。これだけで大半は回る。アクションメソッドの引数型ヒントでも注入される。
- **外部サービス（決済・通知・ストレージ）はインターフェース化して bind**（レベル2）。本番↔テストの差し替えがこれで効く。
- **`singleton`** は「設定済みで使い回したい／生成コストが高い」もの（HTTPクライアント等）。
- **`bind`** は毎回まっさらが欲しい・状態を持たせたくないもの。
- 設定値や `Request` などの実行時依存は `when()->needs()->give()` で渡す。

## ハマりどころ / アンチパターン
- **`new` で生成して注入を殺す**：`new StripeGateway()` と直書きすると差し替え不能＝テストでモックできない。型ヒント or `app()` 経由に。
- **インターフェースを bind し忘れて `make`**：抽象型は実装を登録しないと解決できず `BindingResolutionException`。
- **循環依存**：A が B を、B が A を要求すると解決が回らずエラー。設計見直し（片方を `app()->make()` で遅延取得など）。
- **`singleton` の状態汚染**：使い回されるので、リクエスト固有データを内部に溜めると次リクエストへ漏れる（特に Octane で致命的）。状態を持たせない。
- **サービスロケータ乱用**：あちこちで `app(X::class)` を呼ぶと依存がコードから見えなくなる。原則はコンストラクタ注入、`app()` は型ヒントできない場所の最終手段。

## 関連
[service_providers.md](./service_providers.md)（どこで bind を書くか）/ [facades.md](./facades.md)（コンテナへの別アクセス方法）
