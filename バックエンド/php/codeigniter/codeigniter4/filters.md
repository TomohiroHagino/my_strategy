# フィルタ（before/after＝ミドルウェア相当）（CodeIgniter 4）

## ひとことで言うと
**フィルタ（Filters）** は、コントローラ実行の**前（before）／後（after）に共通処理を挟む**仕組み。認証チェック・CSRF・ヘッダ付与・ログなどを横断的に処理する、他フレームワークの**ミドルウェア相当**。

## 役割・なぜ必要か
- 「全リクエストで認証を確認」「特定グループにCSRF」など**横断的関心事をコントローラから分離**するためにある。各コントローラに同じコードを書く重複を避ける。
- before で**リクエストを止めて別レスポンスを返す**（未認証→ログインへリダイレクト）、after で**レスポンスを加工**（共通ヘッダ追加）できる。

---

## フィルタの実体：Filter インターフェース
```php
// app/Filters/AuthFilter.php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;

class AuthFilter implements FilterInterface
{
    // コントローラの「前」。何か返すとそこで処理を打ち切る
    public function before(RequestInterface $request, $arguments = null)
    {
        if (! session()->get('isLoggedIn')) {
            return redirect()->to('/login');   // ここで中断
        }
        // 何も返さなければ通過
    }

    // コントローラの「後」。レスポンスを加工して返せる
    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        $response->setHeader('X-Frame-Options', 'DENY');
        return $response;
    }
}
```
- **before で値を return すると以降（コントローラ含む）を実行せず即応答**。認可ガードの定番。
- **after はレスポンス確定後**に走る。ヘッダ付与・整形・ログに使う。

## 登録場所：app/Config/Filters.php
```php
// app/Config/Filters.php
public array $aliases = [
    'csrf'         => \CodeIgniter\Filters\CSRF::class,
    'toolbar'      => \CodeIgniter\Filters\DebugToolbar::class,
    'honeypot'     => \CodeIgniter\Filters\Honeypot::class,
    'invalidchars' => \CodeIgniter\Filters\InvalidChars::class,
    'auth'         => \App\Filters\AuthFilter::class,   // 自作にエイリアス
];

// 全リクエストに適用（グローバル）
public array $globals = [
    'before' => [
        'csrf',                                   // 状態変更系を保護
        // 'honeypot',
        // 'invalidchars',
    ],
    'after' => [
        'toolbar',                                // 開発時のデバッグツールバー
    ],
];

// HTTPメソッド単位
public array $methods = [
    // 'post' => ['csrf'],
];

// URIパターン単位
public array $filters = [
    'auth' => ['before' => ['admin/*', 'mypage/*']],
];
```
- **`$aliases`** でクラスに短い名前を付け、**`$globals`/`$methods`/`$filters`** で適用範囲を決める。
- **必ずこの `Config/Filters.php` に登録する**（ここに書かないと、クラスを作っても動かない）。

## 組込フィルタ
- **`csrf`**（CSRF）：POST等の状態変更リクエストでトークン検証。フォームに `csrf_field()` を出す（`security.md` 参照）。
- **`honeypot`**：人間に見えない囮フィールドを仕込み、Botの自動送信を弾く。
- **`invalidchars`**：制御文字・不正なエンコードを含む入力を拒否。
- **`toolbar`**：開発時のデバッグツールバー（本番では外す）。
- **`secureheaders`**：セキュリティ用レスポンスヘッダを付与。

## ルート単位での適用
```php
// app/Config/Routes.php
$routes->group('admin', ['filter' => 'auth'], function ($routes) {
    $routes->get('dashboard', 'Admin::dashboard');
});

// 単一ルートに引数付きで（$arguments で受け取れる）
$routes->get('reports', 'Reports::index', ['filter' => 'auth:editor,admin']);
```
- **グローバル（全体）**＝`Filters.php` の `$globals`、**部分適用**＝`$filters`（URIパターン）またはルート定義の `'filter' => ...`。`auth:role` の `role` 部分は `before()` の `$arguments` に渡る。

## 実行順と before/after の使い分け
```
[globals.before] → [methods.before] → [filters/route.before]
        → Controller →
[filters/route.after] → [methods.after] → [globals.after]
```
- **before は「入る前の検査・中断」**（認証・CSRF・IP制限）。**after は「出る前の加工」**（ヘッダ・ログ・整形）。
- 同種が複数あると**配列の並び順に実行**。認証より先にCSRFを通したい等、順序を意識して並べる。

## ハマりどころ / アンチパターン
- **登録場所を忘れる**：`App\Filters` にクラスを作っても、`Config/Filters.php` の `$aliases` ＋ 適用先（`$globals`/`$filters`/ルート）に書かないと動かない。
- **before/after の取り違え**：リダイレクトや中断は **before**。after で return しても処理は止められない（既にコントローラ実行済み）。
- **実行順の誤解**：globals → methods → route の順。after は逆順。ガードの前提（ログイン状態など）が後段で崩れないよう順序を確認。
- **CSRFを全体に付けてAPIで詰まる**：トークンを送れないAPIには `$filters` で範囲を絞るか、API用に除外設定する。
- **重い処理をbeforeに詰める**：全リクエストで走るので、DB多投・外部API呼び出しは避け、必要な範囲（URIパターン）に限定する。

## 関連: [routing.md](./routing.md) / [security.md](./security.md)
